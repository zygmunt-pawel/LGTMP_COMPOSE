# lgtmp_compose

A standalone, batteries-included **Grafana LGTMP observability stack** in
docker compose. One command, six containers, four pillars of observability,
all wired together with bidirectional cross-signal navigation in Grafana.

```text
   L oki      ←─ logs                              ┌─────────────┐
   G rafana    ─ UI                                │   YOUR APP  │
   T empo     ←─ traces                            │  (any OTel- │
   M imir     ←─ metrics                           │ instrumented│
   + Pyroscope (CPU profiles)                      │  service)   │
   + Alloy    (OTLP collector that fans out to the └──────┬──────┘
              right backend)                              │
                                                          │ OTLP gRPC :4317
                                                          │ OTLP HTTP :4318
                                                          │ Pyroscope :4040
                                                          ▼
                                            ┌──────────────────────┐
                                            │  alloy → loki/tempo/ │
                                            │           mimir      │
                                            │  pyroscope (direct)  │
                                            │                      │
                                            │  grafana :3001 (UI)  │
                                            └──────────────────────┘
```

## Quick start

```bash
git clone https://github.com/zygmunt-pawel/LGTMP_COMPOSE.git
cd LGTMP_COMPOSE
docker compose up -d
```

Open Grafana at <http://localhost:3001> — anonymous Admin (dev defaults).

Point your application's OTLP exporter at:

| Signal | Endpoint | Protocol |
|---|---|---|
| Logs / Traces / Metrics | `localhost:4317` | OTLP gRPC |
| Logs / Traces / Metrics | `localhost:4318` | OTLP HTTP |
| Profiles | `localhost:4040` | Pyroscope native HTTP |

If you're using the [`rust_telemetry`](https://github.com/zygmunt-pawel/rust_telemetry)
crate, this is one `Builder::new() ... .init()` away. Other languages have
their own SDKs; any OpenTelemetry-compliant exporter works.

## What's inside

| Service | Port (host) | What it does |
|---|---|---|
| **alloy** | `4317`, `4318`, `12345` | OTLP collector. Receives logs/traces/metrics from your app; fans them out to loki/tempo/mimir; UI on `:12345`. |
| **loki** | `3100` | Log database. Stores OTLP logs from alloy with structured metadata. |
| **tempo** | `3200` | Trace database. OTLP gRPC ingest internal; query API on `:3200`. metrics_generator + local-blocks for TraceQL Metrics. |
| **mimir** | `9009` | Metric database. PromQL-compatible (Prometheus protocol) on `:9009/prometheus`. Exemplars enabled. |
| **pyroscope** | `4040` | Continuous profiling backend. Receives push from your app's pyroscope agent. |
| **grafana** | `3001` | UI. All four datasources auto-provisioned; example RED dashboard included. |

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
zero changes. Otherwise, edit the queries in the JSON to match your label
schema.

To add your own dashboards, drop them in `grafana/dashboards/` — they're
auto-loaded.

## Common operations

```bash
# Start everything
docker compose up -d

# Tail logs
docker compose logs -f

# Restart a single service after config change
docker compose restart alloy

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

### Putting alloy on a separate machine

If the observability backends run on a dedicated "ops" host while the app
runs elsewhere, the recommended pattern is **alloy as a sidecar with the
app**, exporters pointing at the remote backends. The app pushes to its
local `localhost:4317`; alloy forwards over the network with auth.

Adjust `alloy/config.alloy`'s `otelcol.exporter.*` clients to your remote
hosts and set up auth headers via `auth = otelcol.auth.basic...` blocks.

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

## See also

- [`rust_telemetry`](https://github.com/zygmunt-pawel/rust_telemetry) —
  drop-in OTel SDK + Pyroscope setup for Rust apps. One `Builder::new() ... .init()`
  call wires up logs/traces/metrics/profiles to this stack.

## License

[MIT](LICENSE)
