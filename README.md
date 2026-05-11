# Drssed — Monitoring

GLP stack ([Grafana](https://grafana.com), [Loki](https://grafana.com/oss/loki/), [Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/)) for [drssed-api](https://github.com/davidriegel/drssed-api): real-time log aggregation, structured query, and a curated performance dashboard.

---

## Features

- **Zero-config Docker discovery** — Promtail picks up every container on the host via the Docker socket; no per-service config needed.
- **Structured JSON parsing** — request fields (`method`, `path`, `status_code`, `duration_ms`, `endpoint`, `level`, `request_id`) are extracted from the API's JSON log lines.
- **Stable labels** — streams are keyed by `compose_project`/`compose_service` instead of fragile container-name regexes, so renames don't break queries.
- **TSDB-backed Loki** — schema v13 / TSDB, structured-metadata ingestion enabled, compactor-based retention (30 d by default).
- **Curated dashboard** — RPS, error rate, success rate, p50/p95/p99 latency, status-code mix, top endpoints by latency and by volume, live log tail.
- **Hardened compose** — health-gated startup, capped Docker logs, pinned image versions.

---

## Quick Start

### Prerequisites

- Docker + Docker Compose
- [drssed-api](https://github.com/davidriegel/drssed-api) running with `FLASK_ENV=production` (emits structured JSON logs)

### Installation

```bash
git clone https://github.com/davidriegel/drssed-monitoring.git
cd drssed-monitoring
docker compose up -d
```

Grafana is at <http://localhost:3000>. Default credentials `admin` / `admin` — change on first login.

The **Drssed API Monitoring** dashboard is auto-provisioned and reloads from disk every 10 s.

---

## Dashboard

### Overview row (top of dashboard, range-aware)
| Panel | What it shows |
|---|---|
| **Total Requests** | Total request count over the selected time range |
| **Errors** | Count of `level="ERROR"` lines, background-colored when > 0 |
| **Success Rate** | Share of `2xx`/`3xx` responses; red below 90, green ≥ 99 |
| **P95 Response Time** | Average of per-stream p95 over the last 5 min |
| **Current Throughput** | Requests/s over the last minute |

### Performance row
- **Request Rate by Method** — `sum by (method) (rate(...))`, one line per HTTP verb.
- **Response Time Percentiles** — p50 (green), p95 (yellow), p99 (red); each is `avg(quantile_over_time(...))` so high-cardinality endpoint labels don't fan out into dozens of lines.

### Endpoint & status row
- **Status Codes Over Time** — stacked timeseries, 2xx green / 3xx blue / 4xx orange / 5xx red.
- **Status Code Distribution** — donut with the same color scheme.
- **Top 10 Slowest Endpoints (avg ms)** — bar chart, continuous green→red color scale.
- **Top 10 Endpoints by Volume** — bar chart, continuous blue→purple color scale.

### Logs row
- **Live Logs** — full JSON, pretty-printed and line-wrapped.
- **Errors & Warnings** — filtered to `level=~"ERROR|WARNING"`.

Default time range is **last 1 h**, refresh **30 s**, shared crosshair tooltip across panels.

---

## Integration with drssed-api

### JSON logging

Set `FLASK_ENV=production` in `drssed-api/.env`. Production mode emits log lines like:

```json
{
  "timestamp": "2026-05-11T19:04:11.251496Z",
  "level": "INFO",
  "message": "GET /ping → 200 (0.32ms)",
  "endpoint": "main.ping_reachability",
  "method": "GET",
  "status_code": 200,
  "duration_ms": 0.32,
  "request_id": "…"
}
```

Promtail extracts these fields. The `pipeline_stages` are gated on `compose_service="api"`, so non-API containers (mysql, redis) are still shipped as raw lines without futile JSON parsing.

### Automatic discovery

No host-network sharing or shared compose project required — Promtail uses the Docker socket (read-only) to enumerate containers and reads their JSON log files directly from `/var/lib/docker/containers`. Self-logs from `loki`, `promtail`, and `cloudflare-cloudflare-ddns-1` are dropped at the pipeline stage to keep the index small.

---

## LogQL Examples

Run these in **Explore** → Loki datasource.

**All API logs:**
```logql
{compose_project="drssed-backend",compose_service="api"}
```

**Errors only:**
```logql
{compose_project="drssed-backend",compose_service="api"} | json | level="ERROR"
```

**Slow requests (> 500 ms):**
```logql
{compose_project="drssed-backend",compose_service="api"} | json | duration_ms > 500
```

**Request rate per endpoint:**
```logql
sum by (endpoint) (rate({compose_project="drssed-backend",compose_service="api"} | json | endpoint!="" [1m]))
```

**5xx rate:**
```logql
sum(rate({compose_project="drssed-backend",compose_service="api"} | json | status_code >= 500 [5m]))
```

**P99 latency, single line:**
```logql
avg(quantile_over_time(0.99, {compose_project="drssed-backend",compose_service="api"} | json | __error__="" | unwrap duration_ms [5m]))
```

---

## Tech Stack

| Component | Version | Purpose |
|---|---|---|
| Grafana | 11.4.0 | Visualization, dashboard provisioning |
| Loki | 3.3.2 | Log storage (TSDB v13 schema, filesystem chunks) |
| Promtail | 3.3.2 | Docker SD log collection |

---

## Configuration

### Log retention

Retention is enforced by the Loki compactor. Adjust in `loki-config.yaml`:

```yaml
limits_config:
  retention_period: 720h   # 30 days

compactor:
  retention_enabled: true
  compaction_interval: 10m
```

Apply with `docker compose up -d` (no full restart needed; Loki re-reads on container restart).

### Scraping additional services

Promtail already collects every container the Docker socket can see. To drop a noisy one, add it to the existing drop selector in `promtail-config.yaml`:

```yaml
pipeline_stages:
  - match:
      selector: '{container=~"cloudflare-cloudflare-ddns-1|promtail|loki|my-noisy-thing"}'
      action: drop
```

To run JSON extraction for another service, add a second `match` block selecting on its `compose_service`.

---

## Customizing the dashboard

The provisioned dashboard is editable in the UI:

1. Open **Drssed API Monitoring**.
2. Edit panels, click **Save dashboard**.
3. **Share** → **Export** → **Save to file**.
4. Replace `dashboards/drssed-api-dashboard.json` — Grafana picks it up within ~10 s without a restart.

---

## Operations

### Health

Both Loki and Grafana have HTTP health checks defined in compose; Promtail and Grafana wait for Loki to report ready before starting.

```bash
docker compose ps
curl -s http://localhost:3100/ready
curl -s http://localhost:3000/api/health
```

### Verifying ingestion

```bash
curl -s "http://localhost:3100/loki/api/v1/label/compose_service/values"
# {"status":"success","data":["api","grafana","mysql","redis", ...]}

curl -sG "http://localhost:3100/loki/api/v1/query" \
  --data-urlencode 'query=sum(rate({compose_service="api"} | json | __error__="" [1m]))'
```

If `data` is empty for `compose_service`, the API container isn't labeled by Docker Compose — confirm the API is started via `docker compose`, not a bare `docker run`.

### Container log caps

Each container in this stack logs through `json-file` with `max-size: 10m, max-file: 3` — i.e. 30 MB max per container. The Loki data volume (`loki-data`) is the only thing that grows with traffic.

---

## Related

- **Backend API** → [davidriegel/drssed-api](https://github.com/davidriegel/drssed-api)
- **iOS App** → [davidriegel/drssed-ios](https://github.com/davidriegel/drssed-ios)
- **Portfolio** → [davidriegel.dev](https://davidriegel.dev)

---

## Resources

- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [LogQL Query Language](https://grafana.com/docs/loki/latest/query/)
- [Promtail Configuration](https://grafana.com/docs/loki/latest/send-data/promtail/configuration/)
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
