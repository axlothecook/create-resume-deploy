# create-resume-deploy

The production deployment for **Create_Resume** — runs the whole app on a Raspberry
Pi as a Docker Compose stack, reachable at **https://resume.axlothecook.com** via a
Cloudflare Tunnel. CI (GitHub Actions in the app repos) builds the images, pushes
them to GHCR, and rolls this stack automatically on every push to `main`.

## Stack (`docker-compose.prod.yml`)

| Service | Image | Role |
|---------|-------|------|
| `mongo` | `mongo:8` | database (résumé JSON + accounts), data on the `mongo-data` volume |
| `backend` | `ghcr.io/axlothecook/create-resume-backend:latest` | Express API, port 3006 |
| `frontend` | `ghcr.io/axlothecook/create-resume-frontend:latest` | static SPA served by nginx, port 80 |
| `cloudflared` | `cloudflare/cloudflared:latest` | outbound tunnel to Cloudflare's edge |

This stack is self-contained on its own network + its own tunnel — it shares nothing
with the gaming-shop stack on the same Pi.

## First-time bring-up (on the Pi)

```sh
# 1. clone this repo
git clone https://github.com/axlothecook/create-resume-deploy.git ~/create-resume-deploy
cd ~/create-resume-deploy

# 2. create the env file (NOT in git) — copy the template and fill it in
cp .env.example .env
nano .env        # set SESSION_SECRET + TUNNEL_TOKEN (the rest have sane defaults)

# 3. pull the images and start the stack
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
```

## Cloudflare Tunnel routes (Zero Trust dashboard)

On **this project's own tunnel**, add ONE Public Hostname (Service Type = HTTP, URL
= `host:port` with NO scheme):

| Hostname | Service |
|----------|---------|
| `resume.axlothecook.com` | `frontend:80` |

The API has no separate hostname: the frontend's nginx reverse-proxies
`resume.axlothecook.com/api` → `backend:3006` internally (same origin → first-party
session cookie). If an old `resume-api.axlothecook.com` route still exists, delete it.

## Updating

Just push to `main` in the app repos — CI rebuilds the image, pushes to GHCR, and
runs `docker compose pull && up -d` here over Tailscale. To update by hand:

```sh
cd ~/create-resume-deploy
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
```

See `project-notes/DEPLOY.md` for the full setup walkthrough (GitHub secrets, the
tunnel, the backup cron).
