# lawlan.dev

Personal website and blog, built with [Astro](https://astro.build) (content collections + MDX) and
hosted on **GitHub Pages** at [lawlan.dev](https://lawlan.dev).

## Requirements

Tooling is pinned in `mise.toml` and managed with [mise](https://mise.jdx.dev):

```bash
mise install     # installs Bun 1.4.0
bun install
```

Bun is the only runtime needed — Node.js is not required, since `bun run` provides its own `node`
shim for the Astro CLI.

## Commands

| Command | Action |
| --- | --- |
| `bun install` | Install dependencies |
| `bun run dev` | Dev server at `localhost:4321` |
| `bun run build` | Build the static site to `./dist/` |
| `bun run preview` | Serve the built `./dist/` locally |
| `bun run check` | `astro build && tsc` — run before pushing |
| `bun run deps:check` | `bun outdated` — packages with newer versions |
| `bun run deps:audit` | `bun audit` — known vulnerabilities |

## Project structure

- `src/pages/` — file-based routes (`index.astro` is the About page, `blog/`, `projects/`, `rss.xml.js`)
- `src/content/blog/` — blog posts as `.md`/`.mdx`, frontmatter validated by `src/content.config.ts`
- `src/layouts/`, `src/components/` — shared UI; site-wide constants in `src/consts.ts`
- `public/` — static assets copied verbatim, including `CNAME` (the custom domain)
- `mise.toml` — pinned Bun version, used both locally and in CI

## Dependency updates

- **Dependabot** (`.github/dependabot.yml`) opens weekly upgrade PRs against `bun.lock` and for the
  GitHub Actions used in the workflows. Astro and its integrations are grouped into a single PR;
  other minor/patch bumps are grouped; major bumps come as separate PRs.
- Every PR is verified by `.github/workflows/ci.yml` (`bun install --frozen-lockfile && bun run check`).
- `.github/workflows/dependency-report.yml` posts a weekly `mise outdated` / `bun outdated` /
  `bun audit` summary to the workflow run (Actions → Dependency report → job summary). Run it any
  time via **Run workflow**. `mise outdated` is what catches a new Bun release, since Dependabot
  does not track `mise.toml`.
- Locally: `bun run deps:check`, `bun run deps:audit`, and `bun update --interactive` to pick
  upgrades interactively.

## Deployment

Every push to `main` triggers `.github/workflows/deploy.yml`, which installs the pinned Bun via
`mise`, runs `bun run build`, and publishes `dist/` with `actions/deploy-pages`. No manual deploy
step is needed.

One-time setup on GitHub:

1. **Settings → Pages → Build and deployment → Source: GitHub Actions**
2. **Settings → Pages → Custom domain: `lawlan.dev`**, then enable **Enforce HTTPS**
   once the certificate has been issued.
3. DNS for `lawlan.dev` (apex domain) — four `A` records and four `AAAA` records:

   ```
   A     185.199.108.153
   A     185.199.109.153
   A     185.199.110.153
   A     185.199.111.153
   AAAA  2606:50c0:8000::153
   AAAA  2606:50c0:8001::153
   AAAA  2606:50c0:8002::153
   AAAA  2606:50c0:8003::153
   ```

   Optionally add `CNAME www → lanyitin.github.io` so `www.lawlan.dev` redirects to the apex.

`public/CNAME` keeps the custom domain from being reset on each deploy — do not delete it.
