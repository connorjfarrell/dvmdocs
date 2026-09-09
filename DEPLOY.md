# Deploying the documentation site

The book is a static site (`jupyter-book build dvmdocs/` → `dvmdocs/_build/html/`), so it can
be hosted anywhere that serves static files. Two turnkey options are described below.

> Fork/preview infrastructure: `wrangler.toml`, `.python-version`, and this file can all be
> removed before the upstream pull request.

---

## Cloudflare (recommended)

Best when the domain's DNS is already on Cloudflare — adding a custom subdomain is one click
and HTTPS is automatic. Cloudflare builds the site itself on every push; no CI secrets.

The repo ships [`wrangler.toml`](wrangler.toml) (an assets-only Worker — no server code, just
the static `dvmdocs/_build/html/` output) and [`.python-version`](.python-version).

### One-time setup

1. **Cloudflare dashboard → Workers & Pages → Create → Import a repository** (or *Connect to
   Git*). Select the repo and the production branch (`main`).
2. Fill in the fields:
   | Field | Value |
   |-------|-------|
   | Project name | `dvmdocs` (must match `name` in `wrangler.toml`) |
   | Build command | `pip install -r dvmdocs/requirements.txt && jupyter-book build dvmdocs/` |
   | Deploy command | `npx --yes wrangler@4 deploy` |
   | Builds for non-production branches | ✅ enable (gives every PR a preview URL) |
   | Advanced → non-production branch deploy command | `npx --yes wrangler@4 versions upload` |
   | Advanced → path | `/` |
3. If the build can't find Python / pip, add an environment variable `PYTHON_VERSION` = `3.12`
   (the `.python-version` file should already handle this).
4. Save and deploy. Production is served at
   `https://dvmdocs.<your-workers-subdomain>.workers.dev`; non-production branches get
   `versions upload` preview URLs.

### Custom domain

1. Open the deployed Worker → **Settings → Domains & Routes → Add → Custom Domain** →
   `dvmdocs.wg1gem.com`.
2. Because `wg1gem.com` is on Cloudflare, the `CNAME` record is created automatically and the
   certificate is issued within a minute or two.

No `.nojekyll` needed — Cloudflare does not run Jekyll, so the `_static` / `_images` /
`_sources` directories serve correctly.

---

## GitHub Pages (alternative)

Fully repo-driven via GitHub Actions. Enable once, then every push to `main` redeploys.

### One-time setup

1. Commit `.github/workflows/deploy-pages.yml` (below).
2. **Repo → Settings → Pages → Build and deployment → Source: GitHub Actions.**
3. For the custom domain: **Settings → Pages → Custom domain** → `dvmdocs.wg1gem.com`, then in
   Cloudflare DNS add a `CNAME` `dvmdocs` → `<user>.github.io` (DNS-only / grey cloud is
   simplest; orange-cloud proxying also works). Keep "Enforce HTTPS" checked.

### `.github/workflows/deploy-pages.yml`

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r dvmdocs/requirements.txt
      - run: jupyter-book build dvmdocs/
      - run: touch dvmdocs/_build/html/.nojekyll
      - uses: actions/upload-pages-artifact@v3
        with:
          path: dvmdocs/_build/html

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

The `.nojekyll` file is required here so Pages serves the `_`-prefixed asset directories.

---

## Notes

- This is a **preview/review deployment**. If you share it before the upstream PR is merged,
  make clear it's an unofficial build so people don't mistake it for the canonical
  <https://dvmproject.readthedocs.io/> site.
- Local preview while editing:
  ```bash
  python3 -m venv .venv
  .venv/bin/pip install -r dvmdocs/requirements.txt
  .venv/bin/jupyter-book build dvmdocs/
  python3 -m http.server 8000 --directory dvmdocs/_build/html
  ```
