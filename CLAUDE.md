# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Personal blog built with **Astro 5** (content collections + MDX) and deployed to **Cloudflare Workers** via the `@astrojs/cloudflare` adapter. Requires **Node >= 22**.

## Commands

- `npm run dev` — local dev server at `localhost:4321`
- `npm run build` — build production site to `./dist/` (emits a Cloudflare Worker at `dist/_worker.js/`)
- `npm run preview` — build, then serve the built Worker locally with `wrangler dev`
- `npm run check` — full validation: `astro build && tsc && wrangler deploy --dry-run`. Run this before deploying.
- `npm run deploy` — `wrangler deploy` (expects a prior `npm run build`)
- `npm run cf-typegen` — regenerate `worker-configuration.d.ts` from Wrangler bindings; run after changing bindings in `wrangler.json`
- `wrangler tail` — stream live Worker logs (observability is enabled in `wrangler.json`)

There is no test suite or separate lint step; `tsc` (strict, via `astro/tsconfigs/strict`) is the type check and is included in `npm run check`.

## Architecture

- **Pages** (`src/pages/`) are file-based routes. `blog/[...slug].astro` renders individual posts; `blog/index.astro` lists them; `rss.xml.js` generates the RSS feed. `astro.config.mjs` wires in `mdx()` and `sitemap()` integrations.
- **Content collections**: blog posts live in `src/content/blog/` as `.md`/`.mdx`, loaded via the glob loader in `src/content.config.ts`. Frontmatter is Zod-validated (`title`, `description`, `pubDate` required; `updatedDate`, `heroImage` optional) — adding a field means updating that schema. Retrieve posts with `getCollection('blog')`.
- **Layout/components**: `src/layouts/BlogPost.astro` is the post shell; shared UI (`BaseHead`, `Header`, `Footer`, `FormattedDate`) is in `src/components/`. Site-wide constants (`SITE_TITLE`, `SITE_DESCRIPTION`) are in `src/consts.ts`.
- **Deployment target**: the build produces a Worker (`wrangler.json` `main` → `dist/_worker.js/index.js`) with static assets served from `dist` via the `ASSETS` binding. `nodejs_compat` flag is enabled. `platformProxy` is on in the adapter so Cloudflare bindings work in local dev.

## Notes

- `astro.config.mjs` `site` is set to `https://lawlan.dev`, which drives canonical URLs, sitemap, and RSS output.
- `worker-configuration.d.ts` is generated (do not hand-edit); regenerate via `npm run cf-typegen`.
- This started from Cloudflare's `astro-blog-starter-template`; the sample posts in `src/content/blog/` and default `consts.ts` values are template placeholders.
