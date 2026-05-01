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

## Host + container metrics

LGTMP gives you metrics that **the application itself** emits via OTel
(`http.server.request.duration`, custom counters, etc.). It does not give
you metrics about **the host the app runs on** (CPU, RAM, disk free) or
**the container** (per-container CPU/RAM/network, throttling, OOM proximity)
— those need separate exporters.

Two extra services in [`host-metrics/docker-compose.yml`](host-metrics/docker-compose.yml)
cover this:

| Exporter | Port | What it measures |
|---|---|---|
| **node-exporter** | `:9100` | Whole-host CPU, RAM, disk, network, load avg, swap (reads `/proc`, `/sys`) |
| **cAdvisor** | `:8080` | Per-container CPU, RAM, network, disk I/O, plus **limits** like `container_spec_cpu_quota` and `container_spec_memory_limit_bytes` (talks to docker daemon + cgroups) |

### Pull model — alloy is required

node-exporter and cAdvisor are **pull-only**. They expose `/metrics` on HTTP
and wait for someone to scrape them. They do **not** push to Grafana Cloud
or Mimir directly — that's not in their design.

Alloy bridges this:

```text
                 ┌──────────────────┐
                 │  alloy           │
  /metrics ←─────┤  prometheus.     │  remote_write
                 │   scrape ──────────────→  Mimir / Grafana Cloud
  /metrics ←─────┤                  │
                 └──────────────────┘
```

`prometheus.scrape "node_exporter"` and `prometheus.scrape "cadvisor"` blocks
in `alloy/config.alloy` pull every 15s. `prometheus.remote_write "mimir"`
pushes the result to Mimir's native push API.

### Why pull, why alloy

- Pull discipline (regular sample times, easy "target down" detection via `up == 0`)
- node-exporter / cAdvisor would need HTTPS + auth on every host to push
  directly — pull lets the scraper hold credentials in one place
- Alloy is the standard way to do this in the Grafana stack; alternatives
  (Prometheus standalone, OpenTelemetry Collector, Vector) work the same way

You **cannot** make node-exporter or cAdvisor push directly to Grafana Cloud.
You always need a scraper-pusher in the loop.

### Locality requirement

`/proc`, `/sys`, `/var/run/docker.sock` are **local filesystem things**.
node-exporter and cAdvisor must run **on the same host** as the resources
they observe. You can't scrape a remote host's `/proc`. Alloy can be
anywhere (it just needs network reach to the exporters' HTTP endpoints).

### Three deployment scenarios

#### 1. Single host (default) — easiest

Both compose projects on the same machine. Alloy reaches node-exporter and
cAdvisor via the **host gateway** — Docker maps the hostname `node-exporter`
(set by `extra_hosts: ["node-exporter:host-gateway"]` on alloy) to the
host's IP, which is where `host-metrics` exposed ports `:9100` and `:8080`.

Run it:
```bash
docker compose up -d                              # in lgtmp_compose/
cd host-metrics && docker compose up -d           # in host-metrics/
```

Open Grafana on `http://localhost:3001` — `Explore → Mimir` and try
`up{job="node-exporter"}` (should be 1) or
`container_memory_usage_bytes{name="lgtmp-grafana"}` (live RAM use of the
Grafana container itself).

For dashboards, import these from grafana.com:
- [1860 — Node Exporter Full](https://grafana.com/grafana/dashboards/1860)
- [14282 — cAdvisor Compute Resources](https://grafana.com/grafana/dashboards/14282)

#### 2. Multi-host, alloy on obs-box only — simpler but exposes ports

`obs-box` runs `lgtmp_compose`. `app-box` runs your app + `host-metrics`.
Alloy on obs-box scrapes node-exporter / cAdvisor over the network on
app-box.

On `app-box`:
```bash
cd host-metrics && docker compose up -d
# Ports :9100 (node-exporter) and :8080 (cAdvisor) are now reachable
# from anywhere that can talk to app-box on the network.
```

On `obs-box`, override the host mapping in a
`docker-compose.override.yml` next to `lgtmp_compose/docker-compose.yml`:
```yaml
services:
  alloy:
    extra_hosts:
      - "node-exporter:192.168.1.50"   # app-box's IP or hostname
      - "cadvisor:192.168.1.50"
```

Then `docker compose up -d`. Alloy will resolve `node-exporter:9100` to
`192.168.1.50:9100` and scrape it across the network.

**Caveats:**
- Ports `:9100` and `:8080` on app-box are exposed to whatever can reach it
  (your LAN, possibly internet). Firewall them or use a private network.
- node-exporter has no built-in auth — assume that anyone who can reach
  the port can read your host metrics.
- Network jitter / outages between obs-box and app-box show up as gaps
  in metrics. Alloy's prometheus.scrape doesn't buffer (Prometheus pull
  protocol assumes regular reachability).

#### 3. Multi-host, alloy sidecar on every host — the standard production pattern

Each host runs its own alloy alongside the things it monitors. The local
alloy scrapes its host's node-exporter / cAdvisor and the local app's OTLP
export, and `prometheus.remote_write`s upstream — to a central alloy on
obs-box, or directly to Grafana Cloud.

**Why this is preferred:**
- Alloy local buffers to disk on network glitches (no metric loss for short
  outages)
- App pushes OTLP to `localhost:4317` — never knows any cloud secrets
- node-exporter / cAdvisor stay on a private docker network, no exposed
  ports on the host
- Adding a second app-box is just deploying the same sidecar bundle

**Layout (sketch — not in this repo):**
```
app-box/docker-compose.yml:
  - alloy_app   (scrapes localhost:9100, :8080, receives :4317 from app)
  - node-exporter
  - cadvisor
  - <your app>

obs-box/docker-compose.yml = lgtmp_compose:
  - alloy_obs   (receives push from alloy_app via remote_write)
  - loki, tempo, mimir, pyroscope, grafana
```

Or `alloy_app` writes directly to Grafana Cloud Mimir (with auth headers)
and you skip running a central alloy entirely.

This pattern isn't shipped as a third compose project here — when you need
it, copy `host-metrics/docker-compose.yml` and the relevant
`prometheus.scrape` / `prometheus.remote_write` blocks from `alloy/config.alloy`
into a new `app-box-sidecar/` compose alongside your app.

## See also

- [`rust_telemetry`](https://github.com/zygmunt-pawel/rust_telemetry) —
  drop-in OTel SDK + Pyroscope setup for Rust apps. One `Builder::new() ... .init()`
  call wires up logs/traces/metrics/profiles to this stack.

## License

[MIT](LICENSE)
