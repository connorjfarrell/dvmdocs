# Deploying the documentation site

The book is a static site (`jupyter-book build dvmdocs/` → `dvmdocs/_build/html/`), so it can
be hosted anywhere that serves static files. Two turnkey options are described below.

---

## Cloudflare Pages (recommended)

Best when the domain's DNS is already on Cloudflare — adding a custom subdomain is one click
and HTTPS is automatic. Cloudflare builds the site itself on every push; no CI secrets, no
workflow file.

### One-time setup

1. **Cloudflare dashboard → Workers & Pages → Create → Pages → Connect to Git.**
2. Select the repository (e.g. `connorjfarrell/dvmdocs`) and the branch to deploy (`main`).
3. **Build settings:**
   | Setting | Value |
   |---------|-------|
   | Framework preset | `None` |
   | Build command | `pip install -r dvmdocs/requirements.txt && jupyter-book build dvmdocs/` |
   | Build output directory | `dvmdocs/_build/html` |
   | Root directory | *(leave as `/`)* |
4. **Settings → Environment variables → add:** `PYTHON_VERSION` = `3.12`
5. **Settings → Build → Build system version:** ensure **v2** (or later) — v1 only ships an old
   Python.
6. Save and deploy. You get a `https://<project>.pages.dev` URL, plus an automatic preview
   URL for every pull request.

### Custom domain

1. Project → **Custom domains → Set up a domain** → enter `dvmdocs.wg1gem.com`.
2. Because `wg1gem.com` is on Cloudflare, the required `CNAME` record is created automatically
   and the certificate is issued within a minute or two.

Nothing about the output needs `.nojekyll` — Cloudflare Pages does not run Jekyll, so the
`_static` / `_images` / `_sources` directories serve correctly.

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
