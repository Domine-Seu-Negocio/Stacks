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
