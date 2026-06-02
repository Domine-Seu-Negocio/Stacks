# SDK stack (`sdk`)

DOM360 SDK on Docker Swarm: **Rust v2 backend** (`api-v2`) + **frontend**.
**This folder mirrors the live Portainer stack `sdk`** (endpoint `primary`) — it
is the deploy source of truth. Deploy-only: images come from the registry (CI
builds them), so the `build:` blocks from the dev compose are intentionally
omitted here (a git-backed Portainer stack has no build context).

> **GitHub is authoritative.** Never edit this stack in Portainer's editor.
> Change `docker-stack.yml` here, commit, redeploy.

## Services

| Service | Image | Role | Ingress |
|---------|-------|------|---------|
| `api-v2` | `johannalves/sdk:api-v2-rust-dev` | Rust v2 API, `:3001`, runs SQLx migrations on boot | `sdk-dev.…/api`, `wl-front.…/api`, `api-dev.…` |
| `frontend` | `johannalves/sdk:frontend-rust-dev` | Web UI, `:5173` | `sdk-dev.…`, `wl-front.…` |

Three Traefik routers per service: primary host, white-label host
(`PUBLIC_WHITELABEL_HOST`), and a legacy `api-dev.` host. `api-v2` keeps higher
router priority than `frontend` so `/api` wins over the SPA catch-all.
`api-v2` restarts up to 15 times (boot-time migrations can race a cold Postgres).

## Volumes

| Volume | Mount | Purpose |
|--------|-------|---------|
| `vectorizer_kb_data` | `/data/vectorizer_kb` | Knowledge-base data |
| `sdk_branding_assets` | `/home/sdk/branding-assets` | White-label branding assets |

## Prerequisites

```bash
docker network create --driver overlay --attachable network_public   # if missing
```

PostgreSQL reachable at `DB_HOST` (default `db`), database `dom360_db_sdk`,
shared with the agents stack. **Snapshot Postgres before any deploy** — `api-v2`
auto-migrates on startup.

## Deploy (CLI fallback)

```bash
cp .env.example .env && nano .env
set -a && . ./.env && set +a
docker stack deploy -c docker-stack.yml sdk
```

Normal path is the git-backed Portainer stack — see [`../README.md`](../README.md#migrating-an-existing-portainer-stack-to-git).

## Release a new image

Images are env-driven (`API_V2_IMAGE`, `FRONTEND_IMAGE`). Set the new tag/digest
(Portainer Environment tab, or `.env` for CLI), commit if pinned in-file, redeploy.

## Notes

- `AGENT_SHARED_SECRET` must match the DOM360-Agents stack (mutual auth).
- `VITE_*` are baked into the frontend image at build time (CI); runtime values
  here are fallbacks only.
