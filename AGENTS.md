# Sandbox setup notes

This repository is a **UX case study**: at import time it contained only `README.md`.
There is no application source, no build step, and no dependencies — the deliverable the
README describes is a single clickable HTML prototype.

## What runs

`docker-compose.base44.yml` runs one service, `web`: **nginx:alpine** serving the repository
root as a static site on host port **3000**. Nothing is built into an image; the checkout is
bind-mounted, so adding or editing HTML/CSS/JS shows up on reload with no rebuild.

- `nginx.base44.conf` is mounted as the **full main config** (`/etc/nginx/nginx.conf`), not as
  a `conf.d` snippet, because it sets `user root;` — see below.
- `index.html` is a placeholder status page. `nginx.base44.conf` sets
  `index excel-combiner-redesign.html index.html;`, so once the real prototype exists at the
  repository root as `excel-combiner-redesign.html` it is served at `/` automatically and the
  placeholder becomes unreachable.

## Non-obvious quirks

- **Worker user must be root.** The sandbox checkout directory is mode `0700` and owned by
  root, so nginx's default unprivileged `nginx` worker gets `403 Forbidden` on every request.
  Hence the full main config with `user root;`. If the whole stack were rebuilt from a `conf.d`
  snippet instead, `/` returns 403 again.
- **Dotfiles are denied** in the server block (`location ~ /\.`), so `/.git/config` is a 404
  even though `.git` sits inside the served root.
- No environment variables, database, or external credentials are involved. There is nothing
  to declare as a secret.

## How to verify

```bash
docker compose -f docker-compose.base44.yml up -d
docker compose -f docker-compose.base44.yml ps          # web should be (healthy)
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/   # 200
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/.git/config  # 404
```

A change to any served file needs only a browser reload; only edits to
`docker-compose.base44.yml` or `nginx.base44.conf` need
`docker compose -f docker-compose.base44.yml up -d --force-recreate`.
