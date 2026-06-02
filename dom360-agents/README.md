# DOM360-Agents stack

Voice/SDR agents on Docker Swarm: the SDR/Copilot **API**, the **voice worker**,
and a **self-hosted LiveKit** plane (server + SIP + redis). Mirrors
`DOM360-Agents/prod-docker-stack.yml` — this folder is the deploy source of truth.

> **GitHub is authoritative.** Never edit this stack in Portainer. Change
> `docker-stack.yml` here, commit, then `docker stack deploy` (or let CI redeploy).

## Services

| Service | Image | Role | Ingress |
|---------|-------|------|---------|
| `sdr-agent` | `deselles/sdr-agent` (pinned) | SDR/Copilot HTTP API, `:8000` | `agente-dev.domineseunegocio.com.br` |
| `voice-agent` | same image | LiveKit voice worker, 2 replicas, `start-first`, 600s drain | — (worker) |
| `livekit-server` | `livekit/livekit-server` | WebRTC signaling/media, `:7880/:7881`, UDP `50000-50100` | `…/livekit` |
| `livekit-sip` | `livekit/sip` | SIP gateway, UDP/TCP `:5060`, UDP `10000-10100` | — |
| `livekit-redis` | `redis:7-alpine` | LiveKit + SIP coordination, AOF persistence | — |

`voice-agent` runs ≥2 replicas so LiveKit re-dispatches to a spare when one
drains/crashes; `stop_grace_period: 600s` ≥ `--drain-timeout 600` lets active
calls finish before SIGKILL (see DOM360-Agents ADR 002/003).

## Prerequisites

```bash
# Shared ingress overlay (once per cluster — skip if it already exists)
docker network create --driver overlay --attachable network_public

# LiveKit configs as external Swarm configs (bootstrap once; secret-bearing,
# so the real files are git-ignored — only *.example are committed).
cp config/livekit.yaml.example     config/livekit.yaml       # fill real keys
cp config/livekit-sip.yaml.example config/livekit-sip.yaml   # same keys
docker config create dom360_livekit_config     config/livekit.yaml
docker config create dom360_livekit_sip_config config/livekit-sip.yaml
```

Generate LiveKit keys: `docker run --rm livekit/livekit-server generate-keys`.
The `keys:` in `livekit.yaml`, the `api_key/secret` in `livekit-sip.yaml`, and
`LIVEKIT_API_KEY/SECRET` in `.env` must all match.

PostgreSQL (`dom360_db_sdk`) is shared with the SDK stack — deploy/own it there
or alongside; this stack only connects (`DB_HOST=postgres`).

## Deploy

```bash
cp .env.example .env && nano .env          # fill secrets
set -a && . ./.env && set +a               # export for the stack
docker stack deploy -c docker-stack.yml dom360-agents
```

## Release a new image

Pinned by digest in `docker-stack.yml`. To ship a new build:

1. Edit the `image:` digest on `sdr-agent` **and** `voice-agent` (same image).
2. Commit (`feat(agents): bump image to <digest>`).
3. `docker stack deploy -c docker-stack.yml dom360-agents` (or CI).

`voice-agent` rolls `start-first` with `failure_action: rollback` (60s monitor),
so a bad image auto-rolls back without dropping calls.

## Rotate LiveKit config/keys

```bash
docker config create dom360_livekit_config_v2 config/livekit.yaml   # new versioned config
# point the service at it, then drop the old one:
docker service update --config-rm dom360_livekit_config \
  --config-add source=dom360_livekit_config_v2,target=/etc/livekit.yaml dom360-agents_livekit-server
docker config rm dom360_livekit_config
```

(Swarm configs are immutable — you version + swap, never edit in place.)

## Validate before deploy

```bash
docker run --rm -v "$PWD:/w" -w /w mikefarah/yq e '.services | keys' docker-stack.yml
```
