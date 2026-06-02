# Prometheus monitoring stack

Market-standard monitoring for the DOM360 Docker Swarm: **Prometheus**
(scrape + TSDB + alerting), **Alertmanager** (routing), **Grafana**
(dashboards), **node-exporter** (host metrics, global) and **cAdvisor**
(per-container metrics, global).

## Layout

```
prometheus/
├── docker-stack.yml                     # Swarm stack
├── .env.example                         # Grafana creds + UI basic-auth
├── prometheus/
│   ├── prometheus.yml                   # scrape config (Swarm SD)
│   └── rules/
│       ├── voice-agent.alerts.yml       # voice/SIP alerts (ADR 002)
│       ├── infra.alerts.yml             # node/redis/target alerts
│       └── recording.rules.yml          # precomputed node utilization
├── alertmanager/
│   └── alertmanager.yml                 # routing + inhibition + Slack
└── grafana/
    └── provisioning/
        ├── datasources/datasource.yml   # Prometheus datasource
        └── dashboards/dashboards.yml    # dashboard provider
```

## Prerequisites

```bash
# Shared ingress network (once per cluster)
docker network create --driver overlay --attachable network_public

# Slack webhook secret for Alertmanager
printf '%s' 'https://hooks.slack.com/services/XXX/YYY/ZZZ' \
  | docker secret create alertmanager_slack_url -
```

## Deploy

```bash
cp .env.example .env && edit .env          # GF_ADMIN_PASSWORD, PROM_BASICAUTH
set -a && . ./.env && set +a               # export for the stack
docker stack deploy -c docker-stack.yml monitoring
```

UIs (behind Traefik + basic-auth where noted):
- Grafana — https://grafana.domineseunegocio.com.br
- Prometheus — https://prometheus.domineseunegocio.com.br (basic-auth)
- Alertmanager — https://alertmanager.domineseunegocio.com.br (basic-auth)

## How a service opts into scraping

No Prometheus restart needed — discovery is dynamic via Swarm SD. Add deploy
labels to any service on `network_public`:

```yaml
deploy:
  labels:
    prometheus.enable: "true"
    prometheus.port: "9090"      # the service's metrics port
    prometheus.job: "voice-agent"
    prometheus.path: "/metrics"  # optional, defaults to /metrics
```

## voice-agent metrics contract (DOM360-Agents Phase C 3.3)

The alerts in `rules/voice-agent.alerts.yml` expect the worker to expose:

| Metric | Type | Meaning |
|--------|------|---------|
| `sip_registration_up` | gauge 0/1 | SIP trunk registered (catches the false-healthy mode) |
| `worker_restarts_total` | counter | worker restarts (crash-loop detection) |
| `voice_active_calls` | gauge | in-flight calls (capacity) |
| `voice_stt_latency_seconds` / `_llm_` / `_tts_` | histogram | pipeline latencies |

Until that endpoint exists, the `up`-based alerts (VoiceAgentDown,
VoiceAgentNoHealthyWorkers, TargetDown, RedisDown) already work once the
voice-agent and redis-exporter carry the `prometheus.*` labels.

## Grafana dashboards

Import community dashboards by ID (Dashboards → Import): node-exporter `1860`,
cAdvisor `14282`. Custom dashboards: drop JSON into a volume mounted at
`/var/lib/grafana/dashboards` (provider already configured).

## Validate before deploy

```bash
docker run --rm -v "$PWD/prometheus:/p" prom/prometheus:v2.55.1 \
  promtool check config /p/prometheus.yml
docker run --rm -v "$PWD/prometheus/rules:/r" prom/prometheus:v2.55.1 \
  promtool check rules /r/*.yml
```
