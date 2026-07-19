# Deploy stack for Create Resume repo
This repo runs the Resume Creator project on my Raspberry Pi. It consists of config files that describe how to run the 4 Docker containers together. The instruction files are `docker-compose.prod.yml` and `.env.example`.

## Docker containers
<ul> 
	<li>database: [MongoDB](https://www.mongodb.com)</li> 
	<li>backend: Express API (from GHCR)</li> 
	<li>frontend: the static app served by nginx (from GHCR)</li> 
	<li>cloudflared: [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-tunnel/)</li> 
</ul>


## What does each container do
### MongoDB
It stores user accounts and their saved resumes as documents. Its data lives in a named volume, so a redeploy does not wipe it, and it restarts on its own if it crashes.

### The backend
It runs the API image pulled from GHCR, reads its config and secrets from the .env file, and waits for the database container to start before it does.

### The frontend
It runs the site image pulled from GHCR: an nginx that serves the built static app and proxies `/api` requests to the backend, so everything lives on one domain. It is the only container the tunnel ever reaches, and no ports are published on the Pi at all.

### Cloudflare Tunnel
It runs Cloudflare's tunnel client, which dials out to Cloudflare. That is how the site's domain reaches the frontend without port forwarding or a static IP on my home network.


## Why no graph here
The runtime graph for this stack already lives in the [umbrella README](https://github.com/axlothecook/Create_Resume-umbrella/blob/main/README.md). It shows exactly the containers this repo runs and how they connect, so this README doesn't repeat it.


## The config
Since I cannot commit .env to git, and the real values live only in the Pi's .env, the .env.example lists what the Pi needs: the `SESSION_SECRET` for signing login sessions and the Cloudflare TUNNEL_TOKEN. The rest have working defaults.


## Backups
A cron job on the Pi runs a nightly mongodump (compressed archive) and keeps the last 5 days of archives. Each archive also gets copied off the Pi to my PC, so an SD card failure cannot take the data with it. The backup script lives on the Pi itself, not in this repo.


## Deployment
Handled by my shared [CI/CD pipeline](https://github.com/axlothecook/homelab-ci-cd): a push to the frontend or the backend repo runs that repo's tests, builds the arm64 image, and the Pi pulls it and restarts the stack. If any test fails, nothing gets deployed.


## Tech stack
[Docker Compose](https://docs.docker.com/compose/): runs the 4 containers as one stack, with no ports published on the Pi at all <br />
[nginx](https://nginx.org): lives inside the frontend image; serves the built static app and proxies `/api` to the backend, so everything is one domain; its config file lives in the [frontend repo](https://github.com/axlothecook/Create_Resume) <br />
[GitHub Actions](https://github.com/features/actions) + GHCR (arm64): build and store the images <br />
[Tailscale](https://tailscale.com): the private connection CI uses to reach the Pi <br />
[Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-tunnel/): connects the Pi to the internet outbound-only, so no port forwarding and no static IP <br />
[mongodump](https://www.mongodb.com/docs/database-tools/mongodump/): the nightly database backups


## Where this sits in the pipeline's evolution
This CI/CD deploy pipeline is the second version of my pipeline's evolution. Compared to V1 both repos deploy automatically, the frontend is plain static files behind nginx instead of a Node server, everything lives behind one domain because of the same-origin cookie fix this project forced, and no ports are published on the Pi. 

What it does not have yet: image pruning after deploys, per-service scoped deploys, and a database healthcheck before the backend starts. The test gates arrived last. I recently brought them back to frontend and backend repos of this project.
