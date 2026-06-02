# DOM360-Agents stack (`sdr`)

Voice/SDR agents on Docker Swarm. **This folder mirrors the live Portainer stack
`sdr`** (endpoint `primary`) — it is the deploy source of truth. The voice plane
is self-hosted LiveKit (server + SIP + redis).

> **GitHub is authoritative.** Never edit this stack in Portainer's editor.
> Change `docker-stack.yml` here, commit, redeploy.

## Services

| Service | Image | Role | Ingress |
|---------|-------|------|---------|
| `sdr-agent` | `johannalves/agents:sdr-rust-dev-*` | SDR/Copilot HTTP API, `:8000` | `agente-dev.domineseunegocio.com.br` (stripprefix `/`) |
| `voice-agent` | same image | LiveKit voice worker (`replicas: 1`, restart `any`) | — (worker) |
| `livekit-server` | `livekit/livekit-server:latest` | WebRTC signaling/media | `…/livekit` (priority 10000) |
| `livekit-sip` | `livekit/sip:latest` | SIP gateway | — |
| `livekit-redis` | `redis:7-alpine` | LiveKit + SIP coordination (AOF) | — |

## Ports (why they're long-form)

LiveKit media/SIP ports are published **per-port with `mode: host`**, not as a
`range:range/udp` short form:

| Service | Ports | Mode | Why |
|---------|-------|------|-----|
| `livekit-server` | `7880/tcp` | ingress | HTTP/WS API (can go through the mesh) |
| `livekit-server` | `7881/tcp`, `50000-50100/udp` | host | RTC/RTP must hit the node directly (NAT/WebRTC) |
| `livekit-sip` | `5060/udp+tcp`, `10000-10100/udp` | host | SIP signaling + RTP, same reason |

`mode: ingress` would load-balance UDP across the routing mesh and break media —
hence the explicit per-port `mode: host` blocks.

## External Swarm configs (bootstrap once)

The LiveKit configs are external Swarm configs (secret-bearing → only `*.example`
are committed):

```bash
cp config/livekit.yaml.example     config/livekit.yaml       # fill real keys
cp config/livekit-sip.yaml.example config/livekit-sip.yaml   # same keys
docker config create dom360_livekit_config_20260514_50100 config/livekit.yaml
docker config create dom360_livekit_sip_config            config/livekit-sip.yaml
```

The stack references these exact names. Configs are immutable — to rotate, create
a new versioned name and `docker service update --config-rm/--config-add`.
Generate keys: `docker run --rm livekit/livekit-server generate-keys`. The
`keys:` in livekit.yaml, `api_key/secret` in livekit-sip.yaml and
`LIVEKIT_API_KEY/SECRET` in the env must all match.

## Prerequisites

```bash
docker network create --driver overlay --attachable network_public   # if missing
```

PostgreSQL (`dom360_db_sdk`) is shared with the SDK stack; this stack only
connects (`DB_HOST=postgres`).

## Deploy (CLI fallback)

```bash
cp .env.example .env && nano .env
set -a && . ./.env && set +a
docker stack deploy -c docker-stack.yml sdr
```

Normal path is the git-backed Portainer stack — see [`../README.md`](../README.md#migrating-an-existing-portainer-stack-to-git).

## Release a new image

Pinned by tag in `docker-stack.yml`. To ship a build: edit the `image:` on
`sdr-agent` **and** `voice-agent` (same image), commit (`feat(agents): bump image
to <tag>`), redeploy.
