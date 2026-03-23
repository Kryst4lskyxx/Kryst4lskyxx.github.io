# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal blog (kryst4lskyxx.github.io) built on the [Fuwari](https://github.com/saicaca/fuwari) Astro blog template. It deploys automatically to GitHub Pages on push to `main`.

**Package manager:** pnpm (enforced via `preinstall` hook — do not use npm or yarn)

## Commands

```sh
pnpm install          # Install dependencies
pnpm dev              # Dev server at localhost:4321
pnpm build            # Production build (astro build + pagefind index)
pnpm preview          # Preview production build locally
pnpm check            # Astro type checking
pnpm lint             # Biome lint + fix (writes to ./src)
pnpm format           # Biome format (writes to ./src)
pnpm new-post <name>  # Scaffold a new post in src/content/posts/
```

## Architecture

### Configuration entry point
- `src/config.ts` — primary blog configuration: `siteConfig`, `navBarConfig`, `profileConfig`, `licenseConfig`, `expressiveCodeConfig`. Edit this to change site title, language, theme color, banner, nav links, and profile.
- `src/types/config.ts` — TypeScript types for all config objects.
- `astro.config.mjs` — Astro integrations (Tailwind, Svelte, Swup, ExpressiveCode, sitemap, icon) and markdown pipeline config.

### Content
- `src/content/posts/` — Blog posts as Markdown/MDX files.
- `src/content/config.ts` — Zod schema for post frontmatter: `title`, `published`, `updated`, `draft`, `description`, `image`, `tags`, `category`, `lang`.
- Post frontmatter `lang` field overrides the site language only for that post.

### Pages & layouts
- `src/pages/[...page].astro` — Paginated home/index.
- `src/pages/posts/[...slug].astro` — Individual post pages.
- `src/pages/archive.astro`, `about.astro` — Static pages.
- `src/layouts/MainGridLayout.astro` — Main two-column grid layout wrapping most pages.
- `src/layouts/Layout.astro` — Base HTML shell.

### Components
- `.astro` files are server-rendered components (static).
- `.svelte` files handle client-side interactivity: `Search.svelte`, `LightDarkSwitch.svelte`, `ArchivePanel.svelte`, `DisplaySettings.svelte`.
- `src/components/widget/` — Sidebar widgets (Profile, Tags, Categories, TOC, etc.).
- `src/components/control/` — Reusable UI primitives (Pagination, ButtonLink, etc.).

### i18n
- `src/i18n/i18nKey.ts` — Enum of all translation keys.
- `src/i18n/languages/` — One file per supported language (en, zh_CN, zh_TW, ja, ko, es, th).
- `src/i18n/translation.ts` — `i18n(key)` helper reads `siteConfig.lang` to return the correct string.

### Markdown pipeline (custom plugins in `src/plugins/`)
- `remark-reading-time` — Injects reading time into frontmatter.
- `remark-excerpt` — Extracts post excerpt.
- `rehype-component-admonition` — Renders `:::note`, `:::tip`, `:::warning`, `:::caution`, `:::important` directives.
- `rehype-component-github-card` — Renders `::github{repo}` directives as repo cards.
- ExpressiveCode plugins: `language-badge`, `custom-copy-button`.

### Styling
- Tailwind CSS with nesting (`postcss-nesting`).
- Code blocks use [Expressive Code](https://expressive-code.com/) — theme is set in `src/config.ts` (`expressiveCodeConfig.theme`). Only dark themes are supported for code blocks.
- Icons come from Font Awesome 6 via `astro-icon` + `@iconify-json/fa6-*`. Browse icons at icones.js.org.

### Deployment
- GitHub Actions workflow (`.github/workflows/deploy.yml`) builds and deploys to GitHub Pages on push to `main`.
- `astro.config.mjs` `site` is set to `https://kryst4lskyxx.github.io`.

## Code Style

- Biome handles linting and formatting. Indent style: **tabs**. Quote style: **double quotes** for JS/TS.
- Biome rules are relaxed for `.svelte` and `.astro` files (`useConst` and `useImportType` are off).
