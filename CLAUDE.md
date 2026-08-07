# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal blog ("stdout") built with [Zola](https://www.getzola.org/), a Rust static site generator. Zola is not installed on the host - everything runs through the official Docker image (`ghcr.io/getzola/zola`), matching how the rest of this homelab is run.

## Commands

Local dev server with live reload (serves at http://localhost:1111):

```bash
docker compose up
```

Build (outputs static files to `public/`, gitignored, deployed separately via Netlify - not committed):

```bash
docker run --rm -u "$(id -u):$(id -g)" -v "$PWD:/app" --workdir /app \
    ghcr.io/getzola/zola:v0.22.1 build
```

Check (validates internal/external links and templates without building):

```bash
docker run --rm -u "$(id -u):$(id -g)" -v "$PWD:/app" --workdir /app \
    ghcr.io/getzola/zola:v0.22.1 check
```

There is no separate test suite, linter, or package manager - `check` is the correctness gate.

## Structure

| Path | Purpose |
|---|---|
| `config.toml` | Site config - title, base URL, markdown/syntax highlighting settings |
| `content/posts/` | Blog posts, one `.md` file each |
| `templates/` | Hand-written Tera templates - no third-party theme |
| `static/style.css` | The entire stylesheet - minimal, dark, monospace, no JS, no web fonts |

Template inheritance: `base.html` (header/nav/footer shell) is extended by `index.html` (homepage, latest 10 posts), `section.html` (post listing), and `page.html` (single post). Keep this chain in mind when changing markup - shared structure lives in `base.html` only.

## Adding a post

Create `content/posts/my-post-slug.md`:

```markdown
+++
title = "My Post Title"
date = 2026-07-26
+++

Post content here, in Markdown.
```

## Conventions

- No JS, no web fonts, no third-party theme - keep additions consistent with that minimalism.
- Syntax highlighting uses the `github-dark` theme for both light and dark (see `config.toml`).
