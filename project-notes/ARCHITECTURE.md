# Create_Resume — Architecture (what we built, how, and why)

A plain-language tour of the whole system, written so you can re-read it later and
understand every moving part. Start here; `DEPLOY.md` is the step-by-step runbook.

**Live:** <https://resume.axlothecook.com> · **API:** <https://resume-api.axlothecook.com>

---

## 1. The big picture in one diagram

```
   Your phone / laptop browser
            │
            │  (1) loads the website            (2) the website's JS calls the API
            ▼                                              │
   resume.axlothecook.com                       resume-api.axlothecook.com
            │                                              │
            └──────────────┬───────────────────────────────┘
                           ▼
                  Cloudflare's edge  (free HTTPS, hides the home IP)
                           │
                           │  one secure OUTBOUND tunnel
                           ▼
                ┌──────────────────────────────────────────┐
                │  Raspberry Pi  (at home)                  │
                │                                           │
                │  cloudflared ── routes by hostname ──┐    │
                │       resume.* ──▶ frontend (nginx :80)   │
                │   resume-api.* ──▶ backend (Express :3006)│
                │                          │                │
                │                          ▼                │
                │                       mongo :27017        │
                │                    (data on a volume)     │
                └──────────────────────────────────────────┘
```

Four containers run on the Pi together (Docker Compose): **frontend, backend,
mongo, cloudflared**. The public reaches them only through the Cloudflare tunnel —
no ports are opened on the home router.

---

## 2. The three code repositories

The project is split into three Git repos (linked from the umbrella repo
`Create_Resume-umbrella`):

| Repo | What it is | Tech |
| --- | --- | --- |
| **Create_Resume** | the frontend — the résumé-builder UI | React 19 + Vite (static SPA) |
| **Create_Resume-backend** | the API — accounts + saved résumés | Express 5 + MongoDB + sessions |
| **create-resume-deploy** | the deployment glue (this repo) | Docker Compose + Cloudflare Tunnel |

Why split them? Each has its own lifecycle, its own Docker image, and its own CI
pipeline. The deploy repo holds nothing secret (only a template `.env`) so it can
be public and cloned straight onto the Pi.

---

## 3. How a real request flows (e.g. "save my résumé")

1. You open **resume.axlothecook.com**. Cloudflare serves it over HTTPS and forwards
   the request down the tunnel to the Pi's **frontend** container (nginx), which
   returns the static React app (HTML/JS/CSS).
2. The React app runs in your browser. When you click **Save**, its JavaScript makes
   a `fetch()` to **resume-api.axlothecook.com** (the API base URL was baked into the
   build — see §5).
3. Cloudflare forwards that to the Pi's **backend** container (Express on :3006).
4. The backend checks your **session cookie**, then writes the résumé JSON to the
   **mongo** container.
5. The response (and a refreshed session cookie) travels back out through the tunnel
   to your browser. Done — no home IP exposed, all over HTTPS.

---

## 4. What each piece is and WHY it's there

| Piece | What it does | Why this choice |
| --- | --- | --- |
| **Docker** | packages each app + its exact environment into an image | "works on my PC" → "runs identically on the ARM Pi". The Pi only needs Docker, not Node. |
| **Docker Compose** | one file declaring all 4 containers, started together | one `up -d` brings the whole site up; services find each other by name on a private network. |
| **nginx (frontend)** | serves the static built SPA | a Vite build is just static files — nginx is the smallest, fastest way to serve them (no Node needed at runtime). |
| **Cloudflare Tunnel** | the Pi dials OUT to Cloudflare; visitors hit Cloudflare | no open ports, home IP hidden, free TLS, works behind home NAT. |
| **GitHub Actions + GHCR** | on every push: build the image, push to GHCR, redeploy the Pi | "I pushed code" → "the Pi updates itself", no manual SSH. |
| **Tailscale** | the private network GitHub's runner uses to SSH the Pi | the Pi has no public SSH; the CI runner joins the tailnet as a throwaway node to reach it. |
| **MongoDB volume** | persistent disk for the database | containers are disposable; the `mongo-data` volume keeps your data across restarts. |
| **`restart: unless-stopped`** | Docker restarts crashed containers / after a reboot | the "always-on" guarantee — the site self-heals on a power cycle. |

---

## 5. The two things that make THIS app different from gaming-shop

Both run on the same Pi with the same pipeline, but two design points differ:

1. **The frontend is a *static* SPA, not a server.** Gaming-shop's SvelteKit ships a
   Node server (`node build`); ours is plain static files served by nginx. So our
   Dockerfile is multi-stage: stage 1 runs `npm run build` (Vite → `dist/`), stage 2
   is just nginx copying that `dist/` in. Smaller, simpler, no Node at runtime.

2. **The API URL is baked in at BUILD time.** A static SPA has no server to proxy API
   calls through, so the browser calls the API directly. The address
   (`https://resume-api.axlothecook.com`) is compiled into the JS bundle via the
   `VITE_API_URL` build arg (set in the frontend's CI workflow). That's why a change
   to the API domain requires a *rebuild*, not just a restart. This also means the
   frontend and backend are on **two subdomains** and talk **cross-origin**, which is
   why the backend sets CORS (`CLIENT_ORIGIN`) + a `SameSite=None; Secure` cookie.

---

## 6. The TLS gotcha we hit (and why the API is `resume-api`, not `api.resume`)

We first tried `api.resume.axlothecook.com` for the API. It failed the TLS handshake.

**Why:** Cloudflare's *free* Universal SSL certificate covers the apex and **one**
level of subdomain (`*.axlothecook.com`). `resume.axlothecook.com` is one level → fine.
But `api.resume.axlothecook.com` is **two** levels (`*.resume.axlothecook.com`), which
the free wildcard does **not** cover → no edge certificate → browsers reject the
connection.

**Fix:** use a one-level name — **`resume-api.axlothecook.com`** — which the wildcard
covers. (The alternative, a paid Advanced Certificate for the 2-level name, wasn't
worth it.) Lesson for future projects: **keep public subdomains one level deep** on
the free Cloudflare plan.

---

## 7. CI/CD — how a code change reaches production

Each app repo has `.github/workflows/deploy.yml`. On every push to `main`:

1. **build-and-push** — GitHub's runner builds the Docker image *for arm64* (the Pi's
   CPU, via QEMU emulation) and pushes it to **GHCR** (`ghcr.io/axlothecook/...`).
2. **deploy-to-pi** — the runner joins the **tailnet** (ephemeral node), SSHes to the
   Pi, and runs `docker compose pull && up -d` in `~/create-resume-deploy`. The Pi
   downloads the new image and recreates only the changed container.

Result: `git push` → ~5 min later the live site is updated, hands-off. Verified
working on both repos.

**The 5 GitHub secrets** that power `deploy-to-pi` (same values as gaming-shop, since
the Pi/tailnet/key are shared): `TS_OAUTH_CLIENT_ID`, `TS_OAUTH_SECRET`,
`PI_TAILSCALE_IP` (`100.97.123.51`), `PI_USER` (`axel`), `PI_SSH_KEY`.

---

## 8. Where config and secrets live (three places, three reasons)

1. **Local `.env` files** (your PC) — for dev only. Gitignored. Untouched by deploy.
2. **GitHub repo Secrets** — what the *pipeline* needs (Tailscale + Pi SSH). Encrypted,
   write-only, referenced as `${{ secrets.NAME }}`.
3. **The Pi's `.env`** (`~/create-resume-deploy/.env`) — what the *running containers*
   need: `MONGO_URI`, `CLIENT_ORIGIN`, `SESSION_SECRET`, `NODE_ENV`, `PORT`,
   `TUNNEL_TOKEN`. Gitignored; lives only on the Pi. The `SESSION_SECRET` was
   generated with `openssl rand -hex 48`; the `TUNNEL_TOKEN` comes from the Cloudflare
   tunnel (re-fetchable from the dashboard if ever lost).

---

## 9. What's NOT here (deliberately)

- **No Cloudflare R2 / object storage.** Résumés are pure JSON stored in Mongo; the
  only image (the auth-screen background) is bundled into the frontend build. So,
  unlike gaming-shop, the backend needs no storage keys.
- **No backup yet** (recommended next step). Résumé/account data in Mongo is the only
  thing not reproducible from Git, so a nightly `mongodump` (like gaming-shop's) is
  the one piece of operational hardening still worth adding.
