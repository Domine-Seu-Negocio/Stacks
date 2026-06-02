# SDK stack

DOM360 SDK on Docker Swarm: the **Rust v2 backend** (`api-v2`) and the
**frontend**. Mirrors `SDK/docker-stack.yml` — this folder is the deploy source
of truth. Deploy-only: images come from the registry (CI builds them), so the
`build:` blocks from the source compose are intentionally dropped here.

> **GitHub is authoritative.** Never edit this stack in Portainer. Change
> `docker-stack.yml` here, commit, then `docker stack deploy` (or let CI redeploy).

## Services

| Service | Image | Role | Ingress |
|---------|-------|------|---------|
| `api-v2` | `johannalves/sdk:api-v2-rust-dev` | Rust v2 API, `:3001`, runs SQLx migrations on boot | `sdk.…/api`, `*.<wl-root>/api`, `api.…` |
| `frontend` | `johannalves/sdk:frontend-rust-dev` | Web UI, `:5173` | `sdk.…`, `*.<wl-root>` (white-label) |

Traefik routing covers three host shapes per service: the primary host, the
white-label `HostRegexp` (`{subdomain}.<PUBLIC_WHITELABEL_ROOT_DOMAIN>`), and a
legacy `api.` host. `api-v2` keeps higher router priority than `frontend` so
`/api` wins over the SPA catch-all.

`api-v2` runs SQLx migrations automatically on startup — no separate migration
job. PostgreSQL (`dom360_db_sdk`) is shared with the DOM360-Agents stack.

## Volumes

| Volume | Mount | Purpose |
|--------|-------|---------|
| `vectorizer_kb_data` | `/data/vectorizer_kb` | Knowledge-base data |
| `sdk_branding_assets` | `/home/sdk/branding-assets` | White-label branding assets |

## Prerequisites

```bash
docker network create --driver overlay --attachable network_public   # if missing
```

PostgreSQL reachable at `DB_HOST` (default `db`) with database `dom360_db_sdk`.

## Deploy

```bash
cp .env.example .env && nano .env          # fill JWT_SECRET, DB_PASSWORD, API keys…
set -a && . ./.env && set +a               # export for the stack
docker stack deploy -c docker-stack.yml sdk
```

## Release a new image

Images are env-driven (`API_V2_IMAGE`, `FRONTEND_IMAGE`). To ship a build, set
the new tag/digest in `.env` (or pin it directly in `docker-stack.yml` for an
auditable git trail), commit, then redeploy. Prefer immutable digests in prod.

## Notes

- `AGENT_SHARED_SECRET` must match the DOM360-Agents stack (mutual auth).
- `VITE_*` are baked into the frontend image at build time (CI); the runtime
  env values here are fallbacks only.
