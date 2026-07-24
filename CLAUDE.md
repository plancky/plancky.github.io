# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal portfolio site for Shashvat Jain (shash.cc), built with **Hugo (extended)**. No theme is used — everything lives in the top-level `layouts/`, `assets/`, and `content/`. Deployed to GitHub Pages via GitHub Actions on every push to `main`.

## Commands

Package manager is **bun** (`bun.lockb`). The dev toolchain requires **Hugo extended** and **Dart Sass** installed on the system (Hugo's `toCSS` pipeline needs Dart Sass).

```bash
bun install          # install JS dev deps (Tailwind/PostCSS/Prettier toolchain)
bun run dev          # hugo server — local dev with live reload
bun run build        # hugo --gc --minify ... — production build into ./public
bun run preview      # hugo server in production mode, minified, no fast render
bunx prettier -w .   # format (config in .prettierrc)
```

There is no test suite. CI (`.github/workflows/hugo.yml`) pins `HUGO_VERSION=0.128.0`; local dev has run against newer (0.143.x) — both `extended`. CI overrides `--baseURL` with the Pages URL, so the `baseURL` in `hugo.toml` only matters for local builds.

## Configuration

Config is split under `config/_default/`:
- `hugo.toml` — build settings. Note: `goldmark.renderer.unsafe = true` (raw HTML in Markdown is allowed), TOC is levels 2–3 and `ordered`, and `build.buildStats.enable = true` emits `hugo_stats.json` (see Styling).
- `params.toml` — author info, `katex = true` (enables KaTeX partial), connect/contact links.
- `menus.en.toml` — main nav (Home / About / Projects / Posts) and footer menu.

## Content model

`content/` sections: `about/`, `posts/`, `projects/`. Projects are nested into `projects/websites/` and `projects/hobby/` sub-sections. Posts and some projects are Hugo **page bundles** (a folder with `index.md` + colocated images).

Project entries are driven entirely by front matter — the body Markdown becomes the card description:
```yaml
build: { render: never }          # renders ONLY as a card, no standalone page
params:
    url: "https://…"              # external link the card points to
    thumbnail: "/projects/imgs/…" # served from content/projects/imgs/
    tags: [nextjs, tailwindcss]
weight: 1                          # ordering within its section
```
`build.render: never` is important: those pages exist only to populate cards via the list template, not as visitable URLs.

## Layouts & rendering flow

Standard Hugo lookup order applies. The shell is `layouts/_default/baseof.html`, which defines a `{{ block "page" . }}` filled by `index.html`, `_default/{list,single}.html`, or section-specific templates (`layouts/about/`, `layouts/posts/`).

- `baseof.html` composes the page from `partials/essentials/` (meta, styles, katex, scripts — most `partialCached`) and `partials/components/` (header, glCanvas, footer). The whole `<body>` uses a **CSS grid layout system** (`.content-grid` / `.full-width` / `.breakout` column tracks defined in `assets/scss/layout.scss` and `input.css`) — add `full-width` or `breakout` classes to opt a child out of the centered content column.
- `_default/list.html` powers the Projects page: for each sub-section it shows the first 3 pages as cards (`partials/components/project_card.html`) plus a "See all" link, then lists `RegularPages`. It passes a hand-built `dict` into the card partial rather than the raw page.
- Reusable partials live in `layouts/partials/components/`; shortcodes (`notice`, `blockquote`, `centerquote`, SVGs) in `layouts/shortcodes/`.

## Styling pipeline (important, and slightly non-obvious)

Styling is **Tailwind + SCSS**, compiled by Hugo, not by a standalone build step.

- The real entry point is `assets/scss/main.scss`. `partials/essentials/styles.html` does: `resources.Get "scss/main.scss" | toCSS` (Dart Sass) → `resources.Concat` → `css.PostCSS` (runs Tailwind + autoprefixer per `postcss.config.js`) → in production also `minify | fingerprint | resources.PostProcess`.
- `main.scss` `@import`s `colours`, `layout`, `typography`, `animation` from `assets/scss/`.
- **Design tokens** (shadcn-style HSL vars: `--background`, `--foreground`, `--accent`, …, and `.dark` overrides) live in **`assets/scss/colours.scss`**. `tailwind.config.js` maps them to color utilities (`bg-background`, `text-accent`, etc.).
- **`assets/input.css` is a near-duplicate of these tokens that is NOT wired into the build** — `styles.html` never imports it. Treat `colours.scss` as the source of truth; edit `input.css` only if you deliberately intend to.
- Tailwind's JIT sees which classes are used via **`hugo_stats.json`** (Hugo's `buildStats` writes it; it's mounted as an asset and listed in `tailwind.config.js` `content`). It's gitignored but regenerated on build — if utility classes appear to be missing after a change, that file being stale/absent is the usual cause. `safelist: ["dark"]` keeps the theme class.
- Fonts: `Inter` (body), `Averia Serif Libre` (display), `Orbitron` (button) — see `tailwind.config.js` and `assets/scss/fonts.scss`; TTFs in `static/css/fonts/`.

Note the toolchain mixes Tailwind v3 (`tailwind.config.js`, `"tailwindcss": "3"`) with the v4 PostCSS bridge (`@tailwindcss/postcss`); keep that in mind before upgrading either.

## JavaScript

JS is bundled by Hugo's esbuild (`js.Build`, IIFE format, fingerprinted in prod) from a single entry `assets/js/main.js`, loaded via `partials/essentials/scripts.html`. `main.js` just imports two modules:

- **`assets/js/random_color.js`** — the theme + accent system, exposed as `window.accentColor`. It (1) toggles light/dark by adding/removing `.dark` on `<body>`, persisted to `localStorage["theme"]` and falling back to `prefers-color-scheme`; and (2) **generates a random accent hue on every page load**, writing `--accent` / `--accent-foreground` CSS vars at runtime (overriding `colours.scss`). Theme changes dispatch a `theme:change` event on `documentElement` and expose `onThemeChange(cb)`. The theme-switch and connect-menu buttons in `header.html` hook into this.
- **`assets/js/shader/setup.js`** — a full-viewport WebGL fragment-shader background rendered onto `#glCanvas` (`partials/components/glCanvas.html`). It `fetch`es the GLSL at `/js/shader/coloured_tiles.frag` at runtime (which is why `scripts.html` `<link rel="preload">`s it), tints it with the current accent color, and re-renders via `accentColor.onThemeChange`. `assets/js/phasors/` is a separate canvas animation (`rotating_phasors`).

Because the shader fetches its `.frag` by URL and the theme JS reads/writes global `window` state, these run only in the browser and are wired together through `window.accentColor` and the `theme:change` event rather than imports.

## Formatting conventions

Prettier with `prettier-plugin-go-template` and `prettier-plugin-tailwindcss`, `tabWidth: 4`, `bracketSameLine: true`. HTML/templates are parsed as `go-template` with bracket spacing — expect `{{ ... }}` (spaced) and attributes broken onto their own lines.
