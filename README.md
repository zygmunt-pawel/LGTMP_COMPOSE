# lgtmp_compose

Self-contained **Grafana LGTMP storage stack** in docker compose. One command,
five containers, four pillars of observability storage, all queryable through
a pre-provisioned Grafana with cross-signal navigation already wired.

This stack is intentionally **just the storage/query layer** — it has no
collector. Apps don't push directly here. The recommended setup is:

```text
   ┌─────────────── app-box ──────────────────┐    ┌────────── obs-box ──────────┐
   │                                          │    │                             │
   │  ┌──────────┐                            │    │  loki      :3100            │
   │  │ your app │ ──OTLP localhost:4317──┐   │    │  tempo     :4317 (gRPC)     │
   │  └──────────┘                        │   │    │            :3200 (queries)  │
   │                                      ▼   │    │  mimir     :9009            │
   │  ┌────────────────────────────────────┐  │    │  pyroscope :4040            │
   │  │ alloy_host_agent (systemd, native) │──┼────┼─►grafana   :3001 (UI)       │
   │  │ - host metrics (node_exporter)     │  │    │                             │
   │  │ - container metrics (cAdvisor)     │  │    └─────────────────────────────┘
   │  │ - OTLP forwarder (from your app)   │  │
   │  └────────────────────────────────────┘  │
   └──────────────────────────────────────────┘
```

The agent on app-box ([`alloy_host_agent`](https://github.com/zygmunt-pawel/alloy_host_agent))
collects host + container metrics, accepts OTLP from local apps on
`localhost:4317`, and forwards everything over the network to the backends
in this stack. For single-machine dev, install both on the same machine —
the agent talks to `localhost:4317` (Tempo) etc. For multi-host LAN /
production, install the agent on every app-box and point it at obs-box.

## Quick start

```bash
git clone https://github.com/zygmunt-pawel/LGTMP_COMPOSE.git
cd LGTMP_COMPOSE
docker compose up -d
```

Open Grafana at <http://localhost:3001> — anonymous Admin (dev defaults).

Then install [`alloy_host_agent`](https://github.com/zygmunt-pawel/alloy_host_agent)
on the machine where your application runs, and point it at the endpoints
this stack exposes:

```bash
# in /etc/default/alloy (single-machine — same host):
REMOTE_PROMETHEUS_URL=http://localhost:9009/api/v1/push
REMOTE_OTLP_ENDPOINT=localhost:4317

# in /etc/default/alloy (multi-machine — obs-box on a sibling host):
REMOTE_PROMETHEUS_URL=http://obs-box.lan:9009/api/v1/push
REMOTE_OTLP_ENDPOINT=obs-box.lan:4317
```

Your application then pushes OTLP to its own `localhost:4317` (the agent),
and the agent forwards everything here.

## What's inside

| Service | Port (host) | What it does | Used by alloy_host_agent for |
|---|---|---|---|
| **loki** | `:3100` | Log database with structured-metadata support and pattern_ingester (Drilldown Patterns). | `OTLP HTTP /otlp/v1/logs` |
| **tempo** | `:3200` (queries), `:4317` (OTLP gRPC), `:4318` (OTLP HTTP) | Trace database. metrics_generator + local-blocks for TraceQL Metrics. | `OTLP gRPC :4317` |
| **mimir** | `:9009` | Prometheus-compatible metric database. PromQL on `:9009/prometheus`, push API on `:9009/api/v1/push`, also OTLP HTTP on `:9009/otlp`. Exemplars enabled. | `Prometheus push /api/v1/push` |
| **pyroscope** | `:4040` | Continuous profiling backend (CPU profiles). Native HTTP push protocol. | `Pyroscope native push :4040` |
| **grafana** | `:3001` | UI. All four datasources auto-provisioned with cross-signal links. Example RED dashboard included. | (used by humans, not the agent) |

7-day retention for profiles, 30-day retention for logs (configurable in the
respective config files).

## Cross-signal navigation (already wired)

Grafana datasource provisioning sets up these click-through links:

- **Logs → Trace.** Loki `derivedFields` extracts `trace_id` from every log
  event and renders it as a clickable link to Tempo.
- **Trace → Logs.** Tempo `tracesToLogsV2` jumps from a span to Loki, filtered
  by `trace_id` and the surrounding time window.
- **Trace → Metric.** Tempo `tracesToMetrics` runs PromQL against Mimir for
  the span's `http.route` (rate + p99 latency).
- **Trace → Profile.** Tempo `tracesToProfiles` opens Pyroscope filtered by
  the span's time window (Pyroscope CPU profile).

The metric → trace direction (exemplars) is intentionally **not** wired
because the Rust OTel SDK 0.31 doesn't implement exemplar reservoirs yet —
the line is commented in `grafana/provisioning/datasources/mimir.yml`,
ready to enable when upstream lands support.

## Example dashboard

`grafana/dashboards/rust-app-red.json` is loaded as an example. It expects
metrics with the OTel HTTP semantic-conventions name (`http.server.request.duration`),
which Mimir's OTLP receiver maps to `http_server_request_duration` (no
`_seconds` suffix in Mimir 2.13). 4 panels:

1. **Request rate per route** — `req/s` per `http_route`
2. **Error rate %** — share of 5xx responses
3. **Latency p50 / p95 / p99 per route** — `histogram_quantile`
4. **Status code distribution** — semantic colors per code class

If your app emits OTel HTTP server metrics with the standard label names
(`http_route`, `http_response_status_code`), the dashboard populates with
zero changes.

For node + container dashboards (the data alloy_host_agent collects), import
from grafana.com:
- [1860 — Node Exporter Full](https://grafana.com/grafana/dashboards/1860)
- [14282 — cAdvisor Compute Resources](https://grafana.com/grafana/dashboards/14282)

To add your own dashboards, drop them in `grafana/dashboards/` — they're
auto-loaded.

## Common operations

```bash
# Start everything
docker compose up -d

# Tail logs
docker compose logs -f

# Restart a single service after config change
docker compose restart loki

# Wipe all data (volumes) and start fresh
docker compose down -v && docker compose up -d

# Stop without losing data
docker compose stop
```

## Tweaks

### Retention

- **Loki:** `loki/loki-config.yml` → `limits_config.retention_period`
  (default 30d). Per-stream retention (e.g. longer for ERROR/WARN, shorter
  for INFO) is shown commented.
- **Tempo:** `tempo/tempo-config.yml` — defaults are fine; for custom values
  see Tempo's `compactor.block_retention` setting.
- **Mimir:** Mimir auto-applies blocks_storage TSDB retention (default 13
  months). Change in `mimir/mimir-config.yml` → `limits.compactor_blocks_retention_period`.
- **Pyroscope:** CLI flag in `docker-compose.yml`:
  `-compactor.blocks-retention-period=168h` (7 days). Bump to `720h` for
  30 days, etc.

### Resource limits

Each backend will happily eat as much RAM as the container gets. For a
single-machine deployment add `deploy.resources.limits` per service:

```yaml
services:
  loki:
    # ...
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '0.5'
```

Pyroscope and Mimir are the heavier ones — give them more (~2G each).
The rest are happy with 512M-1G.

### Grafana auth (production hardening)

Default config is **anonymous Admin** — fine for local dev, **a security
hole** for any deployment exposed to a network. Before exposing port 3001
to anything beyond your laptop:

```yaml
grafana:
  environment:
    # Remove these three:
    # - GF_AUTH_ANONYMOUS_ENABLED=true
    # - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    # - GF_AUTH_DISABLE_LOGIN_FORM=true

    # Add these:
    - GF_SECURITY_ADMIN_USER=admin
    - GF_SECURITY_ADMIN_PASSWORD__FILE=/run/secrets/grafana_admin_password
  secrets:
    - grafana_admin_password
```

Or wire OAuth/LDAP/SAML following the
[Grafana docs](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/).

### Auth between alloy_host_agent and the backends

The exposed ports (`:3100`, `:4317`, `:9009`, `:4040`) accept connections
without authentication. Fine for `localhost` or a trusted LAN. For anything
else:

- Put a reverse proxy (Caddy, nginx, traefik) in front of the stack with
  Basic auth; alloy_host_agent's env vars include `REMOTE_PROMETHEUS_AUTH`
  and `REMOTE_OTLP_AUTH` for that.
- Or use mTLS — both Loki/Tempo/Mimir and alloy support it but it's not
  shipped here.

## Caveats

- **Single-binary backends.** Loki, Tempo, Mimir, Pyroscope all run as
  single-binary in this stack (filesystem storage, in-memory ring). Fine
  for single-machine prod, demos, dev. **Do not** scale horizontally with
  this config — for that you need microservices mode + S3 storage. Each
  product's docs cover the upgrade path.

- **No alerts shipped.** Add your own under
  `grafana/provisioning/alerting/` — Grafana picks them up on next restart.

- **Pyroscope :latest pinning.** Pyroscope's `:latest` tag is generally
  stable for OSS use, but its config schema has changed historically
  (e.g. `pyroscopedb.retention` was removed in favor of a CLI flag).
  Pin to a specific version in production.

- **Tempo pinned to 2.9.2.** Newer Tempo mainline removed the local_blocks
  processor we rely on for TraceQL Metrics. Stay on 2.9.x until that's
  resolved upstream.

- **No collector here.** Alloy / OpenTelemetry Collector / Vector — none
  of these are part of this stack. They belong on the application's host,
  not on the storage box. See [`alloy_host_agent`](https://github.com/zygmunt-pawel/alloy_host_agent).

## See also

- [`alloy_host_agent`](https://github.com/zygmunt-pawel/alloy_host_agent) —
  the systemd-native alloy agent that runs on every machine where your app
  lives. Collects host + container metrics, accepts OTLP from local apps,
  forwards everything to this stack.
- [`rust_telemetry`](https://github.com/zygmunt-pawel/rust_telemetry) —
  drop-in OTel SDK + Pyroscope setup for Rust apps. Configure with
  `endpoint: "http://localhost:4317"` and your app's logs/traces/metrics
  flow through alloy_host_agent into this stack.

## License

[MIT](LICENSE)
