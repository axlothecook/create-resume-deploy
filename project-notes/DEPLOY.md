# Create_Resume — Deploy to the Raspberry Pi

> ✅ **STATUS: LIVE.** Deployed and verified end-to-end on 2026-06-08.
> Site: <https://resume.axlothecook.com> · API: <https://resume.axlothecook.com/api> (same origin, nginx-proxied)
> For a plain-language walkthrough of the whole system, read **`ARCHITECTURE.md`**
> (in this folder) first — this file is the operational reference / runbook.

This mirrors the gaming-shop deploy (same Pi, same domain `axlothecook.com`, same
Tailscale CI-to-Pi channel, same GHCR account). Differences are called out below.

## Architecture

```
  git push main (app repo)
        │
        ▼
  GitHub Actions ──build arm64 image──▶ GHCR (ghcr.io/axlothecook/...)
        │
        └──join Tailscale──▶ SSH to Pi ──▶ cd ~/create-resume-deploy
                                            docker compose pull && up -d
        │
        ▼
  Pi stack: mongo + backend + frontend(nginx) + cloudflared
        │
        ▼
  Cloudflare Tunnel ──▶ resume.axlothecook.com  → frontend:80
                        (nginx serves the SPA and proxies /api → backend:3006)
```

## What differs from gaming-shop

- **Frontend is a static SPA**, not a SvelteKit node server. Its Dockerfile is
  multi-stage `node build → nginx`; the final container runs nginx, not `node`.
- **API base URL is build-time** (`VITE_API_URL`, baked by Vite) — set as a build
  arg in the frontend workflow to the relative path **`/api`** (same-origin; nginx
  proxies it to the backend).
- **No R2 / object storage** — résumés are JSON in Mongo; the only image (auth bg)
  is bundled into the frontend build. The backend needs no storage keys.
- **Own tunnel + own network** — fully isolated from the gaming-shop stack.
- **ONE public hostname** (`resume.*`). The API is the `/api` path on the same host,
  reverse-proxied by the frontend's nginx to `backend:3006` — so it's same-origin
  (first-party `SameSite=Lax` cookie), like gaming-shop's server-side proxy.

## Backend production config (already in code — no changes needed)

`app.js` / `createApp.js` are env-driven:
- `NODE_ENV=production` → `secureCookie: true` (HTTPS-only `SameSite=Lax` cookie;
  same-origin now, so Lax not None) + `trustProxy: true` (correct behind the
  Cloudflare Tunnel + the frontend nginx proxy).
- `MONGO_URI=mongodb://mongo:27017/create_resume` (service name, not localhost).
- `CLIENT_ORIGIN=https://resume.axlothecook.com` → the CORS allow-list (harmless now
  that requests are same-origin; kept for safety).
- `SESSION_SECRET` → sign the session cookie (generate a long random value).
- `PORT=3006`.

## GitHub Secrets (add to BOTH app repos → Settings → Secrets and variables → Actions)

Same values as gaming-shop — the Pi + tailnet + deploy key are shared:

| Secret | Value |
|--------|-------|
| `TS_OAUTH_CLIENT_ID` | Tailscale OAuth client ID |
| `TS_OAUTH_SECRET` | Tailscale OAuth secret (`tskey-client-...`) |
| `PI_TAILSCALE_IP` | the Pi's tailnet IP (gaming-shop note had `100.97.123.51` — re-confirm) |
| `PI_USER` | `axel` |
| `PI_SSH_KEY` | full contents of the CI private deploy key (`~/.ssh/gameshop_ci_key`) |

(The same key/OAuth already work for gaming-shop, so this is mostly copy-paste.)

## Cloudflare Tunnel (new, own tunnel)

1. Zero Trust → Networks → Tunnels → **Create a tunnel** (separate from gaming-shop's
   `axlothecook-pi`). Copy its **token** → into the Pi `.env` as `TUNNEL_TOKEN`.
2. Add ONE **Public Hostname** on this tunnel (Service Type **HTTP**, URL =
   `host:port` with **no scheme**):
   - `resume.axlothecook.com` → `frontend:80`
   - The API needs NO separate hostname: the frontend's nginx proxies `/api` →
     `backend:3006` internally (same origin → first-party cookie).
   - If a `resume-api.axlothecook.com` hostname from the old setup still exists,
     **delete it** (and its orphaned DNS CNAME) — it's no longer used.
   - GOTCHA (from gaming-shop): if you get "record already exists", delete the
     orphaned CNAME in the DNS tab, then re-save the route.

## First bring-up checklist — ✅ ALL DONE (2026-06-08)

- [x] Push both app repos to `main` so CI builds + pushes the images to GHCR.
- [x] Add the 5 GitHub secrets to BOTH repos.
- [x] Create the Cloudflare tunnel (`axlothecook-resume`) + the two hostname routes.
- [x] `git clone` this repo to `~/create-resume-deploy` on the Pi.
- [x] `cp .env.example .env` on the Pi; set `SESSION_SECRET` + `TUNNEL_TOKEN`.
- [x] `docker compose -f docker-compose.prod.yml pull && up -d` — 4 containers up.
- [x] Verified: frontend 200, API 200, CORS ok, signup + session round-trip works,
      CI auto-deploy (push → build → Pi) proven on both repos.

### Day-2 operations (how to run things now it's live)

```sh
# SSH to the Pi (over Tailscale)
ssh -i ~/.ssh/gameshop_ci_key axel@100.97.123.51

# from ~/create-resume-deploy on the Pi:
docker compose -f docker-compose.prod.yml ps              # container status
docker compose -f docker-compose.prod.yml logs -f backend # follow a service's logs
docker compose -f docker-compose.prod.yml pull && \
  docker compose -f docker-compose.prod.yml up -d          # manual update (CI also does this)
docker compose -f docker-compose.prod.yml restart frontend # restart one service
```

To ship a change: just `git push` to `main` in the frontend or backend repo — CI
rebuilds the image and rolls the Pi automatically (no manual step).

## Backup (recommended, mirrors gaming-shop)

Add a nightly Pi cron `mongodump --archive --gzip` of the `create_resume` DB into
`~/create-resume-backups/` (keep last ~5–14), and extend the PC-side
`pull-pi-backups.ps1` to scp these too. Résumé data is the only thing not
reproducible from git, so it's the one thing that must be backed up.
