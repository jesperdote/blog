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

`build` outputs static files to `public/` (gitignored, not committed).

## Deployment

Self-hosted on a BananaPi device via Jenkins (`deploy-blog` pipeline, agent `bananapi`) - see
`Jenkinsfile` and `docker-compose.prod.yml`. The pipeline builds the site and serves `public/`
from an `nginx:alpine` container (config: `nginx.conf`) on host port `8015`; a front-proxy on
the same box routes `https://infdxeta.info/blog/` to it and a Cloudflare Tunnel exposes it
publicly. The stylesheet link is cache-busted (`get_url(..., cachebust=true)`) since Cloudflare
caches `style.css` for 4 hours otherwise.

## Structure

| Path | Purpose |
|---|---|
| `config.toml` | Site config - title, base URL, markdown/syntax highlighting settings |
| `content/posts/` | Blog posts, one `.md` file each |
| `templates/` | Hand-written Tera templates (base/index/section/page) - no third-party theme |
| `static/style.css` | The entire stylesheet - minimal, dark, monospace, no JS, no web fonts |
| `static/klept.ico` | Favicon |
| `nginx.conf` | Production nginx config, mounted into the deploy container |

## Adding a post

Create `content/posts/my-post-slug.md`:

```markdown
+++
title = "My Post Title"
date = 2026-07-26
description = "One or two sentences, written like a punchy hook."
+++

Post content here, in Markdown.
```

`description` powers the site's Atom feed (`/blog/atom.xml`), which the
[whoami portfolio](https://github.com/jesperdote/whoami) fetches to show recent posts - see
`CLAUDE.md` for details.
