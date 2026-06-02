# Stacks — DOM360 infra (Docker Swarm)

Deployable infrastructure stacks, one folder per stack. Market-standard layout:
each stack is self-contained (its `docker-stack.yml`, config files, `.env.example`
and a `README.md`).

## Stacks

| Folder | Purpose | Status |
|--------|---------|--------|
| [`prometheus/`](prometheus/) | Monitoring: Prometheus + Alertmanager + Grafana + node-exporter + cAdvisor | ✅ ready |
| [`dom360-agents/`](dom360-agents/) | Voice/agents stack (API + voice worker + livekit-server/sip/redis) | ✅ ready |
| [`sdk/`](sdk/) | SDK backend (Rust v2) + frontend | ✅ ready |
| `redis/` | Shared Redis (LiveKit SIP coordination + app cache) + redis-exporter | ⬜ TODO |

## GitHub is the source of truth — no Portainer edits

Every stack is deployed **only** from the `docker-stack.yml` committed here.
Do **not** edit stacks in the Portainer UI: manual edits drift from git and get
overwritten on the next deploy. To change anything:

1. Edit the stack file (or `.env` / config template) in this repo.
2. Commit with a conventional message (`feat(sdk): …`, `fix(agents): …`).
3. Redeploy: `docker stack deploy -c docker-stack.yml <name>` (or via CI).

Image releases are git-visible too: bump the `image:` tag/digest (or the
`*_IMAGE` env var) in the same flow.

## Current Portainer environment

Portainer `2.21.4` at `portainerdevchat.domineseunegocio.com.br`, one endpoint
`primary` (Swarm agent, `tcp://tasks.agent:9001`). All existing stacks were
created via the **web editor** (`git=no`) — the migration target is to back the
app stacks below with **this git repo** so changes stop happening in the UI.

| Portainer stack | Maps to | Notes |
|-----------------|---------|-------|
| `sdk` (id 9) | [`sdk/`](sdk/) | dev domains, white-label via `PUBLIC_WHITELABEL_HOST`, `api-v2` restart ×15 |
| `sdr` (id 10) | [`dom360-agents/`](dom360-agents/) | image `johannalves/agents:sdr-rust-dev-*`, per-port `mode: host` media, external livekit configs |
| `monitoring-agent` (id 23) | [`prometheus/`](prometheus/) | partial overlap |
| `redis`, `postgres`, `vectorizer`, `chatwoot_app`, `evolution`, `n8n`, `pgbouncer_*`, `nginx-monitoramento` | — | infra deps, not yet in this repo |

> The committed `docker-stack.yml` files here were reconciled to **exactly match
> the YAML currently deployed in Portainer** (`${VAR}` placeholders + external
> configs, no inline secrets) so the cutover below changes nothing at runtime.

## Migrating an existing Portainer stack to Git

Portainer cannot convert a web-editor stack to a git stack in place — you
**recreate** it as a Repository stack. Per stack (`sdk`, then `sdr`):

1. **Pre-flight**: confirm the repo `docker-stack.yml` equals the deployed YAML
   (already reconciled). Snapshot Postgres before touching `sdk`/`sdr`.
2. **Capture env**: the stack's Environment variables live in Portainer, not in
   git. Keep them — they carry over when you recreate (or re-enter from your
   `.env`). `.env.example` lists every key each stack needs.
3. **Recreate as git stack** (Portainer → Stacks → Add stack → *Repository*):
   - Repository URL: `https://github.com/Domine-Seu-Negocio/Stacks`
   - Reference: `refs/heads/main`
   - Compose path: `sdk/docker-stack.yml` (or `dom360-agents/docker-stack.yml`)
   - Re-add the same Environment variables.
   - Enable **GitOps updates** (polling, e.g. 5m, or webhook) for auto-redeploy.
   - Use the **same stack name** so service names/volumes are preserved.
4. **Verify**: services converge, Traefik routers resolve, voice media flows
   (for `sdr`), then delete the old web-editor stack if Portainer kept a dup.

After cutover, every change is: edit file → commit → Portainer auto-pulls (or
click *Pull and redeploy*). No more manual editor changes.

## Conventions (all stacks)

- **Orchestrator**: Docker Swarm (`docker stack deploy -c docker-stack.yml <name>`).
- **Ingress**: Traefik on the external overlay network `network_public`,
  `entrypoints: websecure`, `tls.certresolver: letsencryptresolver`, hosts under
  `*.domineseunegocio.com.br`.
- **Secrets**: never commit real secrets. Use `.env.example` + Swarm
  `secrets:`/`configs:`. Real `.env` is git-ignored.
- **Pinning**: pin image tags/digests in production (avoid `:latest`).
- **Discovery**: app services opt into Prometheus scraping with deploy labels
  `prometheus.enable=true` + `prometheus.port=<port>` (see `prometheus/README.md`).

## Networks

- `network_public` — external, shared, Traefik-managed ingress + cross-stack
  service discovery. Created once: `docker network create --driver overlay --attachable network_public`.
