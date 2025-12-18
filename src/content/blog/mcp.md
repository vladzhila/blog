---
title: 'Model Context Protocol (MCP)'
description: 'Article about MCP'
pubDate: 'Nov 21 2025'
---

# What MCP Is

MCP (Model Context Protocol), is a relatively new protocol that was [introduced by Anthropic](https://www.anthropic.com/news/model-context-protocol) on November 25th, 2024.

It is an open standard that lets AI applications connect to external tools and data sources in a standardized way. It’s like a universal adapter that allows AI models to interact with databases, APIs, file systems, and other services without needing custom code for each integration.

At its core, MCP solves a practical problem: before MCP, every AI application needed its own custom integration for each external service. If you wanted your AI assistant to access GitHub, Slack, and a database, you'd need to write separate code for each one. MCP changes that by providing a single protocol that works across different tools and AI models.

An MCP server exposes tools (like "search web" or "query database") using a standard format. An MCP client (like Cursor or Claude Code) can then connect to any MCP server and understand what tools are available, even if the developers on both sides have never coordinated before.

# How MCP Works

## Participants

MCP follows a simple client-server architecture:

- **MCP host**: The AI application (for example: Claude Code, Claude Desktop, VS Code).
- **MCP client**: A connection object the host creates for each server. One server = one client.
- **MCP server**: The program that provides tools, resources, and prompts.

**Example:** VS Code acts as an MCP host. When it connects to the Sentry MCP server, it spins up one MCP client for that server. If it also connects to the filesystem server, it spins up a second client. Each client talks to exactly **one** server.

## Local and Remote Servers

Servers can run locally or remotely:

- **Local MCP server** runs directly on your machine and typically communicates with the client over a lightweight transport such as stdio. This setup makes sense when the tools or data the model needs live on your computer or inside your development environment.
- **Remote MCP server** runs on external infrastructure - for example, in the cloud or on internal company services. It usually communicates with the client over HTTP or another network transport. This approach is useful when the tools or data are centralized, shared across a team, or need to be accessed from multiple locations.

## Layers

MCP has two layers:

- **Data layer:** Defines JSON-RPC messages, lifecycle management, tools, resources, prompts, notifications.
- **Transport layer:** Handles how messages travel (stdio or HTTP), including framing and authentication.

The protocol itself stays the same regardless of transport.

## Lifecycle management

MCP client and MCP server negotiate: protocol version, supported features, available primitives, who can do what. This handshake ensures both sides speak the same language.

## Primitives

Primitives define what information and actions a MCP server can expose.

### **Server primitives**

### Tools

Actions the AI app can call. Tools are executable functions provided by the server. The LLM can call them to perform real operations: file reads, API requests, searches, database queries, etc.

**Example:** A GitHub MCP server exposes a tool called `repos_list_open_pull_requests`.

When the user asks: “Show me all open PRs for our mobile app repo.”

The LLM can directly call:

`tools/call:
  name: "repos_list_open_pull_requests"
  arguments:
    repo: "mobile-app"`

The server responds with structured data describing each PR.

### Resources

Context data the AI app can read. Resources give the LLM read-only information it can use for reasoning or planning. This could be file contents, environment settings, database schemas, repo metadata, configuration files, etc.

**Example:** A filesystem MCP server exposes a resource:

`resource: "/workspace/package.json"`

If the user asks: “Does this project use TypeScript?”

The AI app can call resources/read on /workspace/package.json, look at the "devDependencies", and then answer accurately using the real data rather than making assumptions.

**Another example:** A database MCP server provides a resource called `db_schema` containing all table and column definitions. The LLM uses it to write correct queries without guessing column names.

### Prompts

Templates that guide LLM interactions. Prompts are reusable templates hosted by the server. They help the client generate consistent LLM instructions for using the server’s tools.

**Example:** A vector-search MCP server includes a prompt called `semantic_query_guide` with:

- instructions for how to phrase searches
- examples of valid and invalid queries
- suggestions for fallback behavior

When the LLM needs help constructing a vector-search request, it can fetch this prompt using prompts/get, fill in the user’s query, and then generate a properly shaped tool call.

**Another example:** An analytics server exposes a prompt called `sql_safe_query_template` that includes:

- patterns for writing safe SQL
- allowed operations
- examples of sanitized queries

The LLM uses this prompt whenever it constructs SQL queries through the server’s tools.

### **Client primitives**

### Sampling

The MCP Server asks the host LLM to generate text. The server can request a model completion without directly calling an LLM itself.

**Example:** Imagine a database MCP server offering a "smart query builder" tool. The user gives a vague request: _"Find all customers who ordered something big last year."_ The server needs a clean SQL query but doesn’t want to ship an LLM inside the server. So the server sends a sampling request to the host: “Given this natural-language request, rewrite it into a safe SQL query.” The host LLM returns something like:

`SELECT * FROM orders WHERE size = 'large' AND year = 2024;`

The server uses that query to run the tool call.

### Elicitation

The MCP Server asks the user for more information or confirmation. If the server lacks required input or wants to avoid risky operations without confirmation, it can request user input through the client.

**Example:** A filesystem MCP server offers a `delete_file` tool. When the LLM tries something like “Delete /Users/username/Documents/2023-report.pdf”. The server doesn’t want to blindly delete files. It sends an elicitation back to the AI app: “The server needs user confirmation: Are you sure you want to delete this file?”

The user sees a prompt in their IDE or client UI:

> Do you want to delete 2023-report.pdf?
>
> [Yes] [No]

The user’s answer is passed to the server, and only then the server continues.

### Logging

The MCP server sends logs to the client for debugging and visibility. Servers use logging to report what they’re doing, warn about errors, or provide progress updates.

**Example:** A cloud-based S3 MCP server is uploading a large file. To keep the user informed, the server emits logs like:

- “Upload started…”
- “Uploaded 40%…”
- “File too large, attempting multipart upload…”
- “Upload complete.”

These messages show up in the AI app’s logging panel or debug console, helping the user understand what’s happening and diagnose issues.

### Notifications

MCP servers can send real-time updates without waiting for a request.

**Example:** If tools change, the MCP server sends `notifications/tools/list_changed`, and the MCP client fetches the updated list.

## Example Flow

Below is a simple end-to-end flow showing how a MCP client interacts with a MCP server.

### Initialization

MCP client sends: protocol version, supported capabilities, client info. MCP server responds with its own capabilities. If everything matches, the connection is ready. After that, the MCP client sends notifications/initialized. This step ensures both sides know:

- What versions they speak.
- What features they support.
- What primitives they will use.

### Tool Discovery

MCP client requests `tools/list`. MCP server responds with all available tools, including: name, title, description, input schema. The AI app collects tools from **all** MCP servers and merges them into a single registry the LLM can use.

### Tool Execution

When a tool is needed, the MCP client sends `tools/call` with:

1. Exact tool name.
2. Arguments matching the tool’s schema.

MCP server responds with a content array containing structured results. The AI app returns this result back to the LLM, which continues the conversation using the fresh data.

### Real-time Updates

If the MCP server adds or removes tools, it sends `notifications/tools/list_changed`. The MCP client then fetches a new list with `tools/list`.

This keeps the AI app’s tool registry up to date without polling.

# Using MCP in Practice

Getting started with MCP depends on whether you're **using it as a consumer** or **building your own server**.

## As a consumer

You choose an MCP-enabled client, connect it to one or more MCP servers, and the client gives the LLM access to new tools. Once the connection is set up, the LLM can call tools just like any other part of your workflow.

### Picking an MCP Client

You start on the client side. The most common options are: Claude Desktop, Claude Code, Cursor, VS Code, Codex, or any other MCP-enabled app from the list at: [https://modelcontextprotocol.io/clients](https://modelcontextprotocol.io/clients**).

Each client handles setup a bit differently, but the idea is always the same - you declare which MCP servers you want to connect.

### Finding MCP Servers

Browse MCP servers here:

https://modelcontextprotocol.io/examples

https://github.com/modelcontextprotocol/servers

### Adding Servers in Your Client

Each MCP server page usually includes integration instructions for specific clients.

For example, Playwright MCP has a clear "Getting started" section:

https://github.com/microsoft/playwright-mcp#getting-started

### What Happens After You Connect

Once the client is configured:

1. The client launches or connects to the server.
2. The server advertises its tools and resources.
3. The client exposes those tools to the LLM.
4. The model starts calling the tools whenever it needs them in your workflow.

There’s nothing to manage manually. The client handles all communication, the server handles all execution, and the model simply uses whatever tools you’ve made available.

These are the kinds of prompts you can use once the connection is active. The model automatically routes the requests to the right server:

- Use _Context7 MCP_ to fetch the latest React documentation for useTransition, then implement a React component that shows a loading state while heavy work runs
- Get the newest React Router docs using _Context7 MCP_ and rewrite my routing setup so it follows the recommended patterns from v7
- Use _GitHub MCP_ to fetch issue #142. Summarize the problem, then generate the code changes needed to fix it
- Use _GitHub MCP_ to fetch my PR number #154, read all code review comments, and suggest the code changes needed in the codebase
- Use _Sentry MCP_ to fetch the most recent production error. Summarize the stack trace and show what line in our code is failing
- Use _Sentry MCP_ to fetch the latest production error. Then use _GitHub MCP_ to locate the file and line that failed, and propose a patch that fixes the issue
- Use _Figma MCP_ to fetch the latest design frame for the Dashboard screen and generate React components that match the visual structure
- Use _Postgres MCP_ to show me all accounts created in the last 24 hours and generate a report of active vs inactive users

**Note:** You don’t always need to say something like “use the GitHub MCP” because the LLM often knows which tool to call without being told explicitly. But when many MCPs are connected, it can pick the wrong one or forget that a specific MCP tool is available. To avoid that, it’s safer to name the tool explicitly.

## Building your own MCP

You can build your own MCP servers to connect an LLM with your data sources, internal services, or databases. This gives the LLM direct access to the tools and information that matter in your workflow.

Creating an MCP server is straightforward with the official SDKs. Whether you're using TypeScript, Python, or another language, the documentation provides clear examples to help you get started:

https://modelcontextprotocol.io/docs/sdk

# When to Use MCP

**Use MCP when** you need to give AI access to multiple external services and want to avoid building custom integrations for each one. If you're connecting to services that already have MCP servers (like GitHub, Slack, or Sentry), you can leverage existing work instead of starting from scratch.

For example, use _Context7 MCP_ to give LLM direct access to the latest documentation for the libraries you use. Or use _Atlassian MCP_ to let LLM retrieve Jira tickets or Confluence pages automatically without copying and pasting anything manually into the chat panel. Or _Figma MCP_ to let LLM inspect design files, extract them, and implement UI components that match your actual design specs.

**Important:** Having too many active MCPs can quickly fill up the context window, so it’s important to monitor your context usage carefully. When there’s not enough free space left, the model’s performance can drop - results may become less accurate or start to hallucinate more.

**Use MCP when** you're building tools for non-technical users who need AI capabilities but can't write code. The abstraction layer helps by making complex APIs accessible through natural language.

**Use MCP when** building internal tools where the standardization helps teams share integrations. One team can build an MCP server for your internal systems, and other teams can use it in their AI applications without needing to understand the underlying complexity.

# Downsides and Limitations

## Context Bloat

MCP servers push their tool definitions straight into the model’s context. A rich server like GitHub can take 50,000+ tokens just to describe its tools. Add a few more servers and you’re burning 60,000-70,000 tokens before the model even reads your prompt.

This leads to:

- Higher costs because you're paying for massive input.
- Slower responses because models have more text to process.
- Worse reasoning because LLMs perform best with small, focused toolsets, not dozens of options.

The recent article from the Anthropic team highlights these downsides:

https://www.anthropic.com/engineering/code-execution-with-mcp

## Performance Overhead

Every tool call interrupts generation:

1. Model stops.
2. Tool runs.
3. Tool output gets inserted back into context.
4. Model starts generating again.

If a task requires multiple steps - search, read a file, write somewhere else - each step grows context even more. Latency stacks up quickly.

## Tool Calling Feels Unnatural for Models

Models were trained on real code from millions of projects, not on synthetic tool-call examples.

That shows up in practice:

- Models hallucinate tool syntax.
- They ignore the tool entirely.
- They choose the wrong tool when too many are available.
- They format parameters incorrectly.
- They loop in the wrong direction.
- Too many tools overwhelm the model.

MCP calls depend on probabilistic inference, which means less stability when it matters.

So that’s why it’s better to explicitly name the MCP server in the prompt, for example:

_"use the context7 MCP to retrieve the latest documentation"_.

## Security and Trust Risks

Letting an AI take actions always increases the attack surface.

Key risks include:

- Destructive actions if the model misunderstands instructions.
- Sandboxing requirements when models execute code. If you don't isolate execution, the model could access the host system in ways you didn't expect.
- Prompt injection, especially when servers fetch untrusted content (docs, webpages, etc.).
- Third-party servers receiving your data and potentially leaking it.

Cloudflare and others now run all model-executed code in tight sandboxes with whitelisted APIs because the risks are real.

# Comparing MCP to Other Approaches

## MCP vs Function Calling APIs

OpenAI, Anthropic, and other providers offer their own function calling features in their APIs. These are actually quite similar to MCP - you define functions with schemas, and the model learns when to call them. The key difference is that function calling APIs are model-specific, while MCP aims to be universal.

## MCP vs Skills

Anthropic recently introduced [Skills](https://www.claude.com/blog/skills) - folders containing markdown instructions and scripts that tell models how to do tasks. Skills are simpler than MCP, requiring no protocol knowledge or server setup. You just drop markdown files in a directory. Yet they can be powerful because they leverage the thing models do best: reading text and writing code.

## MCP vs Code Generation

Cloudflare and others have found that instead of having models call MCP tools directly, it's much more effective to have models write TypeScript or Python code that calls those tools. The code then runs in a secure sandbox. It’s interesting because models are trained on massive amounts of real code but very little tool calling syntax. They're naturally much better at writing code. When you present the same capabilities as a TypeScript API instead of MCP tool definitions, models use 98% fewer tokens and perform significantly better. This "code mode" approach also lets models chain operations efficiently. Instead of calling a tool, waiting for the result, calling another tool, and repeating (with tokens compounding each time), the model writes code that handles the entire workflow and only returns the final result.

Read more about it here: https://blog.cloudflare.com/code-mode/.

# Best Practices

**Keep MCP sets small and focused**. Models perform significantly worse when they have too many options. If you notice quality dropping, check how many MCPs your model has access to. Remove unused ones, or keep them disabled by default and enable them only when your current task needs a specific tool.

**Scope MCPs appropriately**. Use user-level scope for personal MCPs you need across projects. Use project-level scope for team-shared MCPs that should be version-controlled.

**Monitor context usage**. Use tools like the `/context` command in Claude Code to see how much context each MCP server is consuming.

**Review MCP permissions regularly**. Use approval flows for tools that can modify data. Many MCP clients let you require manual approval before certain operations execute. Also give only the minimum access to the MCP tools they actually need, for example allowing an Atlassian MCP server to read Jira issues but blocking permissions to create, update, or delete them.

**Sometimes overkill.** Not every task needs an MCP server. Sometimes simple copy-paste works better and faster.

**Do not use MCP** just because it’s trendy. Use it only when your tasks truly need it. A simpler setup is often better.

# Popular MCP Servers

[GitHub](https://github.com/github/github-mcp-server) - Work with repositories, review pull requests, create issues, and handle code operations

[Playwright](https://github.com/microsoft/playwright-mcp) - Provides browser automation capabilities

[Context7](https://github.com/upstash/context7) - Search documentation across libraries and frameworks

[Atlassian](https://github.com/sooperset/mcp-atlassian) - Connects to Jira and Confluence

[Sentry](https://docs.sentry.io/product/sentry-mcp/#claude-code) - Helps monitor errors, read stack traces, and analyze production issues

[Figma](https://github.com/GLips/Figma-Context-MCP) - Gives the AI access to design files so it can generate code or review UI details

[Slack](https://github.com/modelcontextprotocol/servers-archived/tree/main/src/slack) - Provides channel management and messaging capabilities

[Sequential Thinking](https://github.com/modelcontextprotocol/servers/tree/main/src/sequentialthinking) - Dynamic and reflective problem-solving through thought sequences

[Serena](https://github.com/oraios/serena) - Saves context between sessions so the AI remembers what you were working on

Explore more MCP servers:

https://modelcontextprotocol.io/examples

https://github.com/modelcontextprotocol/servers

# Final Notes

MCP represents an important step in making AI applications more capable, but it still has its own limitations. The protocol standardizes the basic mechanics of how AI connects to external tools. This consistency is valuable - it makes it easier for tool providers and AI applications to communicate using a shared format.

# Useful Sources

Model Context Protocol (MCP) documentation:

https://modelcontextprotocol.io/docs/getting-started/intro

Code execution with MCP: Building more efficient agents (Anthropic):

https://www.anthropic.com/engineering/code-execution-with-mcp

Code Mode: the better way to use MCP (Cloudflare):

https://blog.cloudflare.com/code-mode/
