# ksajan.github.io

Personal site built with Hugo and the Sans theme, customized for a portfolio + writing hub.

## Features
- Hero focused on scaling AI systems from prototype to production
- Projects and writing sections with list/card toggle
- Inline fuzzy search on Writing with tag filters
- Embedded resume page with download link
- Dark mode support

## Quick start
```bash
hugo server -D
```
Then open the local URL from the terminal.

## Build
```bash
hugo --minify
```
Outputs to `public/`.

## Content
- Projects: `content/projects/`
- Writing: `content/posts/`
- About: `content/pages/about.md`
- Resume: `content/pages/resume.md` + `static/resume.pdf`

## Theme
Sans theme vendored in `themes/sans` (no submodule).

## Search
Writing page uses inline fuzzy search with tag filters; requires `layouts/posts/list.html` and `/posts/index.json` (generated via Hugo JSON output).

## CI
GitHub Action builds on push/PR (`.github/workflows/hugo.yml`).
