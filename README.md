# Homelab Log Stack — Loki + Vector + Grafana

A lightweight log ingestion and visualization stack for homelabs, supporting
GELF over TCP and structured syslog over UDP.

## Architecture

```
Application (GELF TCP) ──┐
                          ├──▶  Vector  ──▶  Loki  ──▶  Grafana
Router (Syslog UDP)    ──┘
```

| Component | Image | Purpose |
|-----------|-------|---------|
| Vector | timberio/vector:0.36.0-alpine | Ingests and normalises log streams |
| Loki | grafana/loki:2.9.4 | Log storage and query engine |
| Grafana | grafana/grafana:10.3.3 | Visualisation |

## Directory layout

```
.
├── docker-compose.yml
├── loki-config.yml
├── vector.toml
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── loki.yml
        └── dashboards/
            ├── provider.yml
            └── homelab-logs.json
```

## Quick start

```bash
docker compose up -d
```

Grafana will be available at http://localhost:3000 (admin / admin).

## Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 12201 | TCP | GELF ingestion |
| 12202 | UDP | GELF ingestion (alternative) |
| 5140 | UDP | Syslog ingestion |
| 3100 | TCP | Loki HTTP API |
| 3000 | TCP | Grafana UI |

## Sending logs

### GELF over TCP (port 12201)

GELF messages are newline-delimited JSON. Send with netcat:

```bash
printf '{"version":"1.1","host":"myapp","short_message":"Hello","level":6,"facility":"app"}\n' \
  | nc -q1 localhost 12201
```

GELF level values:

| Level | Meaning |
|-------|---------|
| 0 | emerg |
| 1 | alert |
| 2 | crit |
| 3 | err |
| 4 | warning |
| 5 | notice |
| 6 | info |
| 7 | debug |

Most GELF client libraries (Python, Go, Java, etc.) support TCP mode.
Make sure to configure them to use **newline framing** (not null-byte framing).

### Syslog over UDP (port 5140)

Point your router's remote syslog destination to:

- **Host**: IP address of the machine running Docker
- **Port**: `5140`
- **Protocol**: UDP
- **Format**: RFC 3164 or RFC 5424

Test with the `logger` command:

```bash
logger -n 127.0.0.1 -P 5140 --udp "Test message from router"
```

## Querying logs in Grafana

Go to **Explore** (compass icon) and use LogQL:

| Query | Description |
|-------|-------------|
| `{job="gelf"}` | All GELF messages |
| `{job="syslog"}` | All syslog messages |
| `{job="gelf", host="myapp"}` | GELF from a specific host |
| `{job="syslog", level="err"}` | Syslog errors only |
| `{job="syslog"} \|= "firewall"` | Syslog lines containing "firewall" |
| `{job=~"gelf\|syslog"}` | All logs from both sources |

## Verifying the pipeline

```bash
# 1. Check all containers are running
docker compose ps

# 2. Check Loki is healthy
curl -s http://localhost:3100/ready

# 3. Send a test GELF message
printf '{"version":"1.1","host":"testhost","short_message":"Hello from GELF","level":6,"facility":"app"}\n' \
  | nc -q1 localhost 12201

# 4. Confirm it arrived in Loki
curl -s "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={job="gelf"}' \
  --data-urlencode 'limit=5' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); [print(v) for r in d['data']['result'] for ts,v in r['values']]"
```

## Dashboards

The stack ships with a pre-provisioned **Homelab Logs** dashboard. If it does
not appear after startup, trigger a reload:

```bash
docker compose restart grafana
```

To export a dashboard you have customised in the UI:

1. Open the dashboard in Grafana
2. Click the **Share** icon → **Export** → **Save to file**
3. Place the downloaded JSON in `grafana/provisioning/dashboards/`
4. Restart Grafana: `docker compose restart grafana`

The dashboard will be re-imported automatically on every startup.

## Stopping and cleaning up

```bash
# Stop containers, keep data
docker compose down

# Stop and wipe all data
docker compose down -v
```
