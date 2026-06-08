# Create_Resume — Deploy to the Raspberry Pi

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
                        api.resume.axlothecook.com → backend:3006
```

## What differs from gaming-shop

- **Frontend is a static SPA**, not a SvelteKit node server. Its Dockerfile is
  multi-stage `node build → nginx`; the final container runs nginx, not `node`.
- **API base URL is build-time** (`VITE_API_URL`, baked by Vite) — set as a build
  arg in the frontend workflow, pointing at `https://api.resume.axlothecook.com`.
- **No R2 / object storage** — résumés are JSON in Mongo; the only image (auth bg)
  is bundled into the frontend build. The backend needs no storage keys.
- **Own tunnel + own network** — fully isolated from the gaming-shop stack.
- **Two public hostnames** (frontend + API) because the SPA calls the API
  cross-origin (gaming-shop's SvelteKit proxied server-side, so it needed only one).

## Backend production config (already in code — no changes needed)

`app.js` / `createApp.js` are env-driven:
- `NODE_ENV=production` → `secureCookie: true` (HTTPS-only `SameSite=None` cookie) +
  `trustProxy: true` (correct behind the Cloudflare Tunnel).
- `MONGO_URI=mongodb://mongo:27017/create_resume` (service name, not localhost).
- `CLIENT_ORIGIN=https://resume.axlothecook.com` → the CORS allow-list.
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
2. Add two **Public Hostnames** on this tunnel (Service Type **HTTP**, URL =
   `host:port` with **no scheme**):
   - `resume.axlothecook.com` → `frontend:80`
   - `api.resume.axlothecook.com` → `backend:3006`
   - GOTCHA (from gaming-shop): if you get "record already exists", delete the
     orphaned CNAME in the DNS tab, then re-save the route.

## First bring-up checklist

- [ ] Push both app repos to `main` once so CI builds + pushes the images to GHCR
      (or build/push manually). The Pi can only `pull` images that already exist.
- [ ] Add the 5 GitHub secrets to BOTH repos.
- [ ] Create the new Cloudflare tunnel + the two hostname routes.
- [ ] `git clone` this repo to `~/create-resume-deploy` on the Pi.
- [ ] `cp .env.example .env` on the Pi; set `SESSION_SECRET` + `TUNNEL_TOKEN`.
- [ ] `docker compose -f docker-compose.prod.yml pull && up -d`.
- [ ] Visit `https://resume.axlothecook.com` — sign up, save a résumé, reload to
      confirm the session cookie + save round-trip work end to end.

## Backup (recommended, mirrors gaming-shop)

Add a nightly Pi cron `mongodump --archive --gzip` of the `create_resume` DB into
`~/create-resume-backups/` (keep last ~5–14), and extend the PC-side
`pull-pi-backups.ps1` to scp these too. Résumé data is the only thing not
reproducible from git, so it's the one thing that must be backed up.
