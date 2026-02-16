# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal blog at [vladzhila.pages.dev](https://vladzhila.pages.dev), built with Astro 5 and deployed on Cloudflare Pages. Uses bun as the package manager.

## Development Commands

- `bun install` - Install dependencies
- `bun dev` - Start development server at localhost:4321
- `bun run build` - Build production site to ./dist/
- `bun preview` - Preview production build locally
- `bun format` - Format code with Prettier
- `bun astro check` - Type-check the project

**Do not run `bun dev` automatically.** The user will test manually.

## Architecture

### Content & Routing

Blog posts live in `src/content/blog/` as Markdown/MDX files. Schema in `src/content.config.ts` requires `title`, `description`, `pubDate`; optional `updatedDate`, `heroImage`. Access via `getCollection('blog')` from `astro:content`.

Routes use the `/writing/` URL prefix, not `/blog/`:
- `src/pages/writing/[...slug].astro` - Individual posts (uses `getStaticPaths()`)
- `src/pages/writing.astro` - Post listing page
- `src/pages/index.astro` - About/landing page
- `src/pages/rss.xml.js` - RSS feed

### Layout System

Two layouts, both in `src/layouts/`:
- **SidebarLayout** - Base layout used by all pages. Renders `<BaseHead>`, the sidebar nav, and `<main>` slot.
- **BlogPost** - Wraps SidebarLayout. Adds post header (title, date), table of contents, clickable heading anchors, and image zoom modal.

### Theme System

Dark/light mode via `data-theme` attribute on `<html>`. Theme is initialized inline in `BaseHead.astro` before render (prevents FOUC), respects `localStorage` then `prefers-color-scheme`. CSS variables in `src/styles/global.css` define both palettes. Shiki code blocks use Catppuccin themes (Latte for light, Macchiato for dark) with manual CSS overrides to follow the `data-theme` toggle rather than Shiki's default `prefers-color-scheme`.

### Key Configuration

- `src/consts.ts` - `SITE_TITLE`, `SITE_DESCRIPTION`
- `astro.config.mjs` - Site URL, MDX + Sitemap integrations, Catppuccin syntax highlighting, `rehype-external-links` (opens external links in new tabs)
- `.prettierrc` - No semicolons, single quotes, trailing commas (es5), 2-space tabs

### Fonts

Merriweather (regular + bold) loaded from `/public/fonts/` as `.woff`, preloaded in `BaseHead.astro`.
