# stdout

Personal blog, built with [Zola](https://www.getzola.org/). No Zola install on the
host - everything runs through the official Docker image
(`ghcr.io/getzola/zola`), matching how the rest of this homelab is run.

## Writing locally

```bash
docker compose up
```

Serves the site at http://localhost:1111 with live reload while editing files
under `content/`.

## One-off commands (build, check)

```bash
docker run --rm -u "$(id -u):$(id -g)" -v "$PWD:/app" --workdir /app \
    ghcr.io/getzola/zola:v0.22.1 build

docker run --rm -u "$(id -u):$(id -g)" -v "$PWD:/app" --workdir /app \
    ghcr.io/getzola/zola:v0.22.1 check
```

`build` outputs static files to `public/` (gitignored - deployed separately via
Netlify, not committed).

## Structure

| Path | Purpose |
|---|---|
| `config.toml` | Site config - title, base URL, markdown/syntax highlighting settings |
| `content/posts/` | Blog posts, one `.md` file each |
| `templates/` | Hand-written Tera templates (base/index/section/page) - no third-party theme |
| `static/style.css` | The entire stylesheet - minimal, dark, monospace, no JS, no web fonts |

## Adding a post

Create `content/posts/my-post-slug.md`:

```markdown
+++
title = "My Post Title"
date = 2026-07-26
+++

Post content here, in Markdown.
```
