# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Astro-based blog built from the Astro blog template. It supports Markdown and MDX content with RSS feed generation, sitemap, and SEO optimization.

## Development Commands

All commands use `bun` as the package manager:

- `bun install` - Install dependencies
- `bun dev` - Start development server at localhost:4321
- `bun build` - Build production site to ./dist/
- `bun preview` - Preview production build locally
- `bun format` - Format code with Prettier
- `bun astro check` - Type-check the project

**IMPORTANT:** Do not run `bun dev` or start the development server automatically. The user will test manually.

## Architecture

### Content Collections

The blog uses Astro's Content Collections API for type-safe content management:

- Blog posts are defined in `src/content/blog/` as Markdown or MDX files
- Content schema is defined in `src/content.config.ts` with zod validation
- Required frontmatter fields: `title`, `description`, `pubDate`
- Optional fields: `updatedDate`, `heroImage`
- Access posts via `getCollection('blog')` from `astro:content`

### Routing & Pages

- `src/pages/blog/[...slug].astro` - Dynamic route for individual blog posts using `getStaticPaths()`
- `src/pages/blog/index.astro` - Blog listing page
- `src/pages/rss.xml.js` - RSS feed generation endpoint
- All pages use file-based routing

### Layouts & Components

- `src/layouts/BlogPost.astro` - Main blog post layout that wraps individual posts
- Layouts receive `CollectionEntry<'blog'>['data']` as props
- Components are standard Astro components in `src/components/`

### Global Configuration

- `src/consts.ts` - Site-wide constants (`SITE_TITLE`, `SITE_DESCRIPTION`)
- `astro.config.mjs` - Astro configuration including site URL, integrations (MDX, Sitemap), and markdown settings (using Nord theme for syntax highlighting)

### Styling

- Uses minimal custom CSS with CSS variables
- Global styles in `src/styles/global.css`
- Component-scoped styles within .astro files
