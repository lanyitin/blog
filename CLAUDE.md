# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Personal blog built with **Astro** (content collections + MDX), output as a fully static site and
hosted on **GitHub Pages** at `https://lawlan.dev`. There is no adapter and no SSR — every route is
prerendered at build time.

## Toolchain

- **Bun** is the package manager and script runner; the version is pinned in `mise.toml` and
  installed with [mise](https://mise.jdx.dev) (`mise install`). CI installs the same version via
  `jdx/mise-action`, so `mise.toml` is the single source of truth.
- **Node.js is not required.** `bun run` supplies its own `node` shim, so the `#!/usr/bin/env node`
  shebang on `node_modules/.bin/astro` resolves without Node installed. `package.json` declares
  `engines.bun`, not `engines.node`.
- Use `bun` / `bun run` in commands and docs — never `npm`, `npx`, or `yarn`. The npm equivalent of
  `npx foo` is `bunx foo`.
- `bun.lock` (text format) is committed; there is no `package-lock.json`.

## Commands

- `bun install` — install dependencies (`--frozen-lockfile` in CI)
- `bun run dev` — local dev server at `localhost:4321`
- `bun run build` — build the static site to `./dist/`
- `bun run preview` — serve the already-built `./dist/` locally
- `bun run check` — full validation: `astro build && tsc`. Run this before pushing.
- `bun run deps:check` (`bun outdated`) / `bun run deps:audit` (`bun audit`)

There is no test suite or separate lint step; `tsc` (strict, via `astro/tsconfigs/strict`) is the
type check and is included in `bun run check`.

## Architecture

- **Pages** (`src/pages/`) are file-based routes. `index.astro` is the About/home page;
  `blog/[...slug].astro` renders individual posts; `blog/index.astro` lists them; `rss.xml.js`
  generates the RSS feed. `astro.config.mjs` wires in `mdx()` and `sitemap()` integrations.
- **Content collections**: blog posts live in `src/content/blog/` as `.md`/`.mdx`, loaded via the
  glob loader in `src/content.config.ts`. Frontmatter is Zod-validated (`title`, `description`,
  `pubDate` required; `updatedDate`, `heroImage` optional) — adding a field means updating that
  schema. Retrieve posts with `getCollection('blog')`.
- **Layout/components**: `src/layouts/BlogPost.astro` is the post shell; shared UI (`BaseHead`,
  `Header`, `Footer`, `FormattedDate`) is in `src/components/`. Site-wide constants (`SITE_TITLE`,
  `SITE_DESCRIPTION`) are in `src/consts.ts`.

## Deployment

- `.github/workflows/deploy.yml` runs on every push to `main`: `mise-action` → `bun install
  --frozen-lockfile` → `bun run build` → `upload-pages-artifact` → `deploy-pages`. GitHub Pages
  source must be set to **GitHub Actions**.
- `public/CNAME` contains `lawlan.dev` and is copied into `dist/` on each build — it is what keeps
  the custom domain configured across deploys. Do not delete or rename it.
- Because the site is served from an apex custom domain, `base` stays unset in `astro.config.mjs`.
  If hosting ever moves to `lanyitin.github.io/blog`, `base` would need to be set to `/blog`.

## Dependency updates

- `.github/dependabot.yml` uses `package-ecosystem: bun` (updates `package.json` + `bun.lock`) plus
  a `github-actions` entry. `astro` + `@astrojs/*` are grouped into one PR (they must move
  together); other minor/patch bumps are grouped; majors arrive as individual PRs.
- `.github/workflows/ci.yml` runs `bun run check` on every pull request — this is what verifies a
  Dependabot PR before merge. Merging to `main` then triggers the deploy workflow.
- `.github/workflows/dependency-report.yml` runs weekly and writes `mise outdated`, `bun outdated`,
  and `bun audit` into the run's job summary. `mise outdated` covers the Bun version itself, which
  Dependabot cannot see.
- Upgrading locally: `bun update` stays inside the declared ranges; `bun update --latest` crosses
  majors; `bun update --interactive` lets you pick. `bun audit fix` patches vulnerable transitive
  deps.

## Notes

- `astro.config.mjs` `site` is set to `https://lawlan.dev`, which drives canonical URLs, sitemap,
  and RSS output.
- GitHub Pages serves static files only — no server-side rendering, redirects, custom headers, or
  API routes. Any feature needing those has to be solved client-side or at the DNS/CDN layer.
