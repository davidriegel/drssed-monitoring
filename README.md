# Drssed — Monitoring

GLP stack for [drssed-api](https://github.com/davidriegel/drssed-api) with real-time log aggregation, metrics visualization, and performance monitoring.

---

## Features

- **Real-time log streaming** — aggregate logs from all API containers
- **Structured log parsing** — extract fields for querying
- **Pre-configured Grafana dashboards** — out-of-the-box API monitoring
- **Zero-configuration setup** — automatic setup for drssed-api

---

## Quick Start

### Prerequisites

- Docker
- Docker Compose
- [drssed-api](https://github.com/davidriegel/drssed-api) running with `FLASK_ENV=production`

### Installation

Clone the repository:

```bash
git clone https://github.com/davidriegel/drssed-monitoring.git
cd drssed-monitoring
```

Start the stack:

```bash
docker compose up -d
```

Access Grafana:

```
http://localhost:3000
```

**Default credentials:**
- Username: `admin`
- Password: `admin`

⚠️ **Change the password on first login**

The pre-configured dashboard **"Drssed API Monitoring"** is automatically available under **Dashboards** → **General**.

---

## Dashboard Overview

The included dashboard provides comprehensive API monitoring:

### Overview Section
- **Total Requests** — Request volume over the last 5 minutes
- **Error Count** — Number of errors with threshold indicators
- **Success Rate** — Percentage of 2xx responses
- **P95 Response Time** — 95th percentile latency

### Performance Section
- **Request Rate by Method** — HTTP method distribution over time
- **Response Time Percentiles** — P50, P95, P99 latency tracking
- **Top 10 Slowest Endpoints** — Average response time by endpoint
- **Status Code Distribution** — HTTP status code breakdown

### Logs Section
- **Live Logs** — Real-time log stream from all API containers
- **Error Logs Only** — Filtered view of ERROR level logs

All panels update automatically every 30 seconds.

---

## Integration with drssed-api

### Enable JSON Logging

The monitoring stack works best with structured JSON logs. In drssed-api, set:

```env
# drssed-api/.env
FLASK_ENV=production
LOG_LEVEL=INFO
```

Production mode automatically outputs JSON-formatted logs:

```json
{
  "timestamp": "2026-03-18T20:47:42.373646Z",
  "level": "INFO",
  "message": "GET /ping → 200 (0.32ms)",
  "endpoint": "main.ping_reachablility",
  "method": "GET",
  "status_code": 200,
  "duration_ms": 0.32,
  "ip": "172.21.0.1"
}
```

### Automatic Discovery

No manual configuration required. The monitoring stack automatically discovers and monitors all Docker containers via the shared `loki` network.

### Log Query Examples

Use these LogQL queries in Grafana Explore for custom analysis:

**All API logs:**
```logql
{container=~".*drssed.*api.*"}
```

**Error logs only:**
```logql
{container=~".*drssed.*api.*"} | json | level="ERROR"
```

**Slow requests (> 500ms):**
```logql
{container=~".*drssed.*api.*"} | json | duration_ms > 500
```

**Requests by endpoint:**
```logql
sum by (endpoint) (rate({container=~".*drssed.*api.*"} | json [5m]))
```

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Visualization | Grafana |
| Log Aggregation | Loki |
| Log Collection | Promtail |

---

## Configuration

### Log Retention

Modify retention period in `loki-config.yaml`:

```yaml
table_manager:
  retention_period: 720h  # 30 days (in hours)
```

Apply changes:
```bash
docker compose down
docker compose up -d
```

### Monitor Additional Services

The stack automatically monitors all Docker containers. To filter specific services, edit `promtail-config.yaml`:

```yaml
scrape_configs:
  - job_name: docker
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*drssed.*|.*your-service.*)'
        target_label: 'container'
```

---

---

## Customization

### Modify the Dashboard

The provisioned dashboard is fully editable:

1. Open **Drssed API Monitoring** dashboard
2. Edit panels as needed
3. Click **Save dashboard**
4. Export via **Share** → **Export** → **Save to file**
5. Replace `dashboards/drssed-api-dashboard.json`

Changes persist across container restarts.

### Add Custom Panels

Create new visualizations using LogQL queries:

1. Click **Add** → **Visualization**
2. Select **Loki** datasource
3. Enter LogQL query
4. Choose visualization type
5. Save panel

## Related

- **Backend API** → [davidriegel/drssed-api](https://github.com/davidriegel/drssed-api)
- **iOS App** → [davidriegel/drssed-ios](https://github.com/davidriegel/drssed-ios)
- **Portfolio** → [davidriegel.dev](https://davidriegel.dev)

---

## Resources

- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [LogQL Query Language](https://grafana.com/docs/loki/latest/logql/)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)
- [Promtail Configuration](https://grafana.com/docs/loki/latest/clients/promtail/configuration/)

---

## About the Project

This monitoring stack demonstrates observability practices for containerized microservices. It showcases automated log aggregation, real-time metrics visualization, and performance monitoring using Grafana and Loki.

Built as part of the Drssed ecosystem to enable data-driven performance optimization and proactive issue detection.

---