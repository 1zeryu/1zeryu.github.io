# Yukun Zhou

Personal site: homepage plus a writing archive, built with plain Hugo (no theme).

## Structure

- `content/_index.md` — homepage intro (avatar + one short paragraph)
- `data/news.yaml`, `data/publications.yaml` — shared data, rendered on `/about/`
- `content/post/<slug>/index.md` — posts (page bundles)
- `content/page/writing/`, `content/page/about/`, `content/page/search/` — standalone pages
- `layouts/index.search.json` — generates `searchindex.json` for client-side search
- `layouts/` — all templates; `_default/baseof.html` is the shell
- `assets/css/main.css` — the only stylesheet (white background, single column)
- `static/css/chroma.css` — generated code highlighting, regenerate with `hugo gen chromastyles --style=github`
- `static/vendor/katex/` — vendored KaTeX, loaded only on pages with `math: true`

## Local preview

```bash
hugo server --disableFastRender
```

Production build: `hugo --gc --minify`.

## Deploy

GitHub Actions builds and publishes `public/` to GitHub Pages on push to `main`.
