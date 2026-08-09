# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal blog ("stdout") built with [Zola](https://www.getzola.org/), a Rust static site generator. Zola is not installed on the host - everything runs through the official Docker image (`ghcr.io/getzola/zola`), matching how the rest of this homelab is run.

## Commands

Local dev server with live reload (serves at http://localhost:1111):

```bash
docker compose up
```

Build (outputs static files to `public/`, gitignored):

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

## Deployment

Self-hosted on a BananaPi device, not Netlify. Jenkins (`deploy-blog` pipeline, see
`Jenkinsfile`) splits build and deploy across two agents: `vps-host` (amd64) runs `zola build`
and stashes `public/`, because the official Zola image has no armv7 build and the BananaPi is
32-bit armv7. `bananapi` then unstashes it and runs `docker-compose.prod.yml` (`nginx:alpine`
serving `public/` on host port `8015`, with `nginx.conf` mounted in - see below) - note it's
the legacy standalone `docker-compose` binary there, not the `docker compose` plugin subcommand
used locally. A separate front-proxy repo (`klept-lab/proj`, `front/nginx/default.conf`)
proxies `https://infdxeta.info/blog/` to that port; a Cloudflare Tunnel on the BananaPi exposes
it publicly. `base_url` in `config.toml` must stay in sync with that path.

`nginx.conf` sets `absolute_redirect off` - without it, nginx's own trailing-slash redirects
(e.g. hitting `/posts` with no trailing slash) come back as absolute URLs built from the
container's own point of view, which has no idea it's mounted under `/blog/` - dropping both
the prefix and the `https` scheme. The front-proxy repo's `proxy_redirect / /blog/;` directive
is the other half of this fix; both sides are required together, see that repo's CLAUDE.md.

The stylesheet link uses `get_url(path="style.css", cachebust=true)` in `base.html`, not a
plain path - Cloudflare caches `style.css` aggressively (`max-age=14400`), so without a
content-hash query string that changes on every edit, a CSS change can deploy successfully and
still be invisible to visitors for hours.

## Structure

| Path | Purpose |
|---|---|
| `config.toml` | Site config - title, base URL, markdown/syntax highlighting settings |
| `content/posts/` | Blog posts, one `.md` file each |
| `templates/` | Hand-written Tera templates - no third-party theme |
| `static/style.css` | The entire stylesheet - minimal, dark, monospace, no JS, no web fonts |
| `static/klept.ico` | Favicon, reused from the `whoami`/devops-profile site for brand consistency |
| `nginx.conf` | Production nginx config for the `docker-compose.prod.yml` container (see Deployment) |
| `templates/atom.xml` | Overrides Zola's built-in feed template to use `page.description` as the entry `<summary>` (falls back to Zola's native `<!-- more -->`-based `page.summary`, then full `page.content`) - see Adding a post |

Template inheritance: `base.html` (header/nav/footer shell) is extended by `index.html` (homepage, latest 10 posts), `section.html` (post listing), and `page.html` (single post). Keep this chain in mind when changing markup - shared structure lives in `base.html` only.

## Adding a post

Create `content/posts/my-post-slug.md`:

```markdown
+++
title = "My Post Title"
date = 2026-07-26
description = "One or two sentences, written like a punchy hook - not a generic summary."
+++

Post content here, in Markdown.
```

**`description` is required, not optional.** Site-wide feed generation is on
(`generate_feeds = true` in `config.toml`, output at `/blog/atom.xml`), and `templates/atom.xml`
is a custom override that uses `page.description` as each entry's `<summary>` - the
[whoami portfolio](https://github.com/jesperdote/whoami) fetches this feed client-side to
populate its own "Recent posts" section, so a missing `description` means a bad-looking entry
over there, not just a bare feed. Without it, the entry falls back to Zola's native
`<!-- more -->`-marker excerpt (`page.summary`) if one exists in the post body, then to the
full rendered post as a last resort - neither of which is what you want for a feed reader or
the portfolio card.

## Conventions

- No JS, no web fonts, no third-party theme - keep additions consistent with that minimalism.
- Syntax highlighting uses the `github-dark` theme for both light and dark (see `config.toml`).
