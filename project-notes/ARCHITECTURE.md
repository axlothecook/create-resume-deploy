# Create_Resume — Architecture (what we built, how, and why)

A plain-language tour of the whole system, written so you can re-read it later and
understand every moving part. Start here; `DEPLOY.md` is the step-by-step runbook.

**Live:** <https://resume.axlothecook.com> · **API:** <https://resume.axlothecook.com/api> (same origin, reverse-proxied)

---

## 1. The big picture in one diagram

```
   Your phone / laptop browser
            │
            │  loads the site AND calls the API — both on ONE domain
            ▼
   resume.axlothecook.com   (the page; the API is the /api path on the same host)
            │
            ▼
                  Cloudflare's edge  (free HTTPS, hides the home IP)
                           │
                           │  one secure OUTBOUND tunnel  (one hostname)
                           ▼
                ┌──────────────────────────────────────────┐
                │  Raspberry Pi  (at home)                  │
                │                                           │
                │  cloudflared ── resume.* ──▶ frontend (nginx :80)
                │                                  │        │
                │                 nginx serves the SPA, and │
                │                 proxies /api ──▶ backend (Express :3006)
                │                                  │        │
                │                                  ▼        │
                │                               mongo :27017│
                │                            (data on a volume)
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
   a `fetch()` to **resume.axlothecook.com/api/...** — the SAME domain as the page
   (the base URL `/api` was baked into the build — see §5).
3. Cloudflare forwards that to the Pi's **frontend** container; nginx sees the `/api`
   prefix and reverse-proxies the request to the **backend** container (Express :3006),
   stripping `/api` so the backend sees `/auth/login`, `/resumes`, etc.
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

2. **The API URL is baked in at BUILD time — and it's a same-origin path.** A static
   SPA has no server of its own, so the API address is compiled into the JS bundle via
   the `VITE_API_URL` build arg (set in the frontend's CI workflow). We set it to the
   **relative path `/api`**, so the browser calls the API on the *same* origin as the
   page; nginx (in the frontend container) reverse-proxies `/api` to the backend.
   Because page and API share one origin, the session cookie is **first-party**
   (`SameSite=Lax`) — no CORS needed, and strict browsers don't block it. A change to
   this path still requires a *rebuild* (it's baked in), not just a restart.

---

## 6. Two gotchas about the API's address (history + the current design)

### 6a. Why the API is now a same-origin `/api` path (the cross-site-cookie fix)

Originally the API lived on its **own subdomain**, `resume-api.axlothecook.com`,
separate from the page on `resume.axlothecook.com`. That made the login session cookie
a **third-party / cross-site cookie** (it had to be `SameSite=None; Secure`). Browsers
with cross-site-tracking protection on — Safari/iOS, Brave, Firefox, increasingly
Chrome — **silently drop** such cookies (or the whole request), so login failed with a
raw `Failed to fetch` on some phones while working fine on permissive devices. It
looked like a per-device bug; it was really the split-domain architecture.

**Fix:** serve the API as a **path of the same domain** — `resume.axlothecook.com/api`
— with nginx reverse-proxying `/api` → `backend:3006`. Same origin → the cookie is
**first-party** (`SameSite=Lax`, the safe default) → no browser blocks it, and CORS is
no longer needed. **Lesson for future projects: for cookie/session auth, keep the
frontend and backend on the SAME origin (an `/api` path), never on separate subdomains.**

One proxy detail that matters: nginx forwards the browser's original scheme to the
backend via `X-Forwarded-Proto` (defaulting to `https`), so the backend — which trusts
the proxy — still marks the cookie `Secure`. Without that, the backend would see the
internal HTTP hop and refuse the Secure cookie.

### 6b. (Historical) the TLS gotcha that shaped the OLD subdomain name

When the API *was* a subdomain, we first tried `api.resume.axlothecook.com` and it
failed the TLS handshake: Cloudflare's *free* Universal SSL covers only **one** level
of subdomain (`*.axlothecook.com`), not two (`*.resume.axlothecook.com`). We worked
around it with the one-level `resume-api.axlothecook.com`. The same-origin `/api`
design in §6a makes this moot (there's no second public hostname now), but the rule
still holds for any *other* public subdomain on the free plan: **keep it one level deep.**

### 6c. Why moving the API needed NO Cloudflare change

A natural question: "if the API moved from `resume-api.*` to `resume.*/api`, why didn't
I have to change anything in the Cloudflare dashboard?" Because **Cloudflare routes by
HOSTNAME, not by path** — and the hostname it needs (`resume.axlothecook.com`) was
already pointed at the Pi from day one.

- Cloudflare's tunnel route `resume.axlothecook.com → frontend:80` forwards the
  **entire** request — whatever the path: `/`, `/assets/...`, or `/api/auth/login` —
  down to the nginx (frontend) container. Cloudflare neither inspects nor cares about
  the `/api` part. So `/api` requests were *already arriving* at nginx; there was just
  nothing handling them before.
- The actual move happened **inside the Pi**, below Cloudflare's view, in two places we
  ship via the normal `git push → CI → Pi` pipeline (no dashboard edit):
  1. **nginx config** gained `location /api/ { proxy_pass http://backend:3006/; }` —
     so nginx now forwards `/api` traffic to the backend. `backend:3006` is an
     *internal* Docker network name that only exists inside the Pi; Cloudflare can't
     see or route to it.
  2. **the frontend build** changed `VITE_API_URL` to `/api`, so the SPA now *calls*
     `resume.axlothecook.com/api/...` (same domain) instead of the old subdomain.
- The OLD design is the only reason Cloudflare ever had an API entry: it needed a
  SECOND hostname route (`resume-api.* → backend:3006`) to reach the backend directly,
  bypassing nginx. That route is now **unused** — which is why the sole remaining
  Cloudflare task is the *optional* deletion of it (see DEPLOY.md). The new design needs
  just the one `resume.*` route that already existed.

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
