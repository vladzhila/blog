# Vlad's Blog

A personal blog about Claude Code, context engineering, Cursor rules, and MCP.

Live at [vladzhila.pages.dev](https://vladzhila.pages.dev)

## Tech Stack

- [Astro 5](https://astro.build) with MDX support
- [Catppuccin](https://catppuccin.com) syntax highlighting (Latte/Macchiato)
- [bun](https://bun.sh) package manager
- Deployed on [Cloudflare Pages](https://pages.cloudflare.com)

## Project Structure

```text
├── public/
├── src/
│   ├── assets/
│   ├── components/
│   ├── content/
│   │   └── blog/
│   ├── layouts/
│   ├── pages/
│   └── styles/
├── src/content.config.ts
├── astro.config.mjs
├── package.json
└── tsconfig.json
```

Blog posts live in `src/content/blog/` as Markdown or MDX files. The content schema is defined in `src/content.config.ts`.

## Commands

All commands are run from the root of the project:

| Command              | Action                                       |
| :------------------- | :------------------------------------------- |
| `bun install`        | Install dependencies                         |
| `bun dev`            | Start dev server at `localhost:4321`          |
| `bun run build`      | Build production site to `./dist/`            |
| `bun preview`        | Preview production build locally              |
| `bun format`         | Format code with Prettier                     |
| `bun astro check`    | Type-check the project                        |
