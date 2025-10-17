<img width="1920" height="1080" alt="gemini_aurora_thumbnail_4g_e74822ff0ca4259beb718" src="https://github.com/user-attachments/assets/0d2d5070-339f-4feb-8c32-2af8f6bed5bc" />

# Gemini CLI Observability

## 1) Overview

- **Gemini CLI → OTel Collector (OTLP)**
    - **Metrics** exposed by the Collector’s Prometheus exporter on **:9464** → **Prometheus scrapes** it → **Grafana** shows metrics. The Prometheus exporter’s default port is **9464**. ([OpenTelemetry](https://opentelemetry.io/docs/specs/otel/metrics/sdk_exporters/prometheus/?utm_source=chatgpt.com))
    - **Logs** forwarded by the Collector via **OTLP/HTTP** to **Loki** at `/otlp` (Loki’s native OTLP log ingest). ([Grafana Labs](https://grafana.com/docs/loki/latest/send-data/otel/?utm_source=chatgpt.com))
- **Grafana** reads from **Prometheus** (metrics) and **Loki** (logs). When Grafana and Prometheus/Loki run in Docker, use **service names** (not `localhost`) for data sources. ([Grafana Labs](https://grafana.com/docs/learning-journeys/prometheus/add-data-source-url/?utm_source=chatgpt.com))

---

## 2) Prerequisites

- Docker & Docker Compose v2+
- Free local ports: **3000 (Grafana), 3100 (Loki), 4317 (OTLP gRPC), 4318 (OTLP HTTP), 9464 (Collector metrics), 9090 (Prometheus)**

---

## 3) Create the project

```bash
mkdir -p ~/gemini-observability
cd ~/gemini-observability

```

Create the files below **exactly** as shown.

Make sure to set DEV_NAME and the PROJECT_NAME in the environment variables. By doing so we 

can filter metrics and logs via User name and the Project Name.

### `docker-compose.yml`

```yaml
version: "3.7"

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    command:
      - --config.file=/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    command: ["--config=/etc/otel-collector-config.yml"]
    volumes:
      - ./otel-collector-config.yml:/etc/otel-collector-config.yml
    environment:
      # Pass WSL env vars into the container so the collector can expand them
      - DEV_NAME=${DEV_NAME}
      - PROJECT_NAME=${PROJECT_NAME}
    ports:
      - "4317:4317"  # OTLP gRPC receiver for traces/metrics/logs (from Gemini CLI)
      - "9464:9464"  # Prometheus scrape endpoint exposed by the Collector
    depends_on:
      prometheus:
        condition: service_started
      loki:
        condition: service_healthy

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/loki-config.yml
    volumes:
      - ./loki-config.yml:/etc/loki/loki-config.yml
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://localhost:3100/ready || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 5s
    restart: unless-stopped

volumes:
  grafana-data:
```

### `loki-config.yml`

```yaml
server:
  http_listen_port: 3100

auth_enabled: false

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  max_structured_metadata_size: 50MB
```

### `otel-collector-config.yml`

```yaml
receivers:
  # OTLP Receiver: listen for gRPC and HTTP
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  # Metrics -> scraped by Prometheus
  prometheus:
    endpoint: "0.0.0.0:9464"
    # Convert resource attributes into Prometheus labels
    resource_to_telemetry_conversion:
      enabled: true

  # Logs -> Loki native OTLP endpoint
  otlphttp/logs:
    endpoint: http://loki:3100/otlp
    tls:
      insecure: true

processors:
  # Add service/developer/project as resource attributes (read from container env)
  resource/add_context:
    attributes:
      - action: upsert
        key: service.name
        value: gemini-cli
      - action: upsert
        key: developer.name
        value: ${DEV_NAME}
      - action: upsert
        key: project.name
        value: ${PROJECT_NAME}
  batch:

service:
  pipelines:
    # (Traces pipeline removed)
    metrics:
      receivers: [otlp]
      processors: [resource/add_context, batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [resource/add_context, batch]
      exporters: [otlphttp/logs]
```

> Loki natively ingests OTLP logs over HTTP; with the OTel Collector, use the otlphttp logs exporter to /otlp. (Grafana Labs)
> 

### `prometheus.yml`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:9464"]
```

> Prometheus scrapes the Collector’s exporter on 9464 (the Prometheus exporter’s default). (OpenTelemetry)
> 

### (Gemini CLI) `settings.json`

```json

{
  "context": {
    "fileName": ["AGENT.md"]
  },
  "mcpServers": {
    "azure": {
      "command": "uv",
      "args": [
        "--directory",
        "/home/damianpeiris/dev/ai-first-tools/mcp_servers/azure_devops_server",
        "run",
        "server.py"
      ],
      "cwd": "/home/damianpeiris/dev/ai-first-tools/mcp_servers/azure_devops_server",
      "timeout": 30000,
      "env": {
        "AZURE_DEVOPS_TOKEN": "${AZURE_DEVOPS_TOKEN}",
        "AZURE_DEVOPS_ORG": "${AZURE_DEVOPS_ORG}"
      }
    }
  },
  "security": {
    "auth": {
      "selectedType": "oauth-personal"
    }
  },
  "telemetry": {
    "enabled": true,
    "target": "local",
    "otlpEndpoint": "http://localhost:4317",
"useCollector" : true,
"otlpProtocol" : "grpc",
"logPrompts" : true
  }
}

```

---

## 4) Start, stop, and inspect

```bash
# start
docker compose up -d

# see all containers
docker compose ps

# follow logs (pick one)
docker compose logs -f otel-collector
docker compose logs -f loki
docker compose logs -f prometheus
docker compose logs -f grafana

# stop
docker compose down
# reset volumes (wipes Grafana state, etc.)
docker compose down -v

```

---

## 5) Open the UIs

- **Grafana:** [http://localhost:3000](http://localhost:3000/) (default admin/admin)
- **Prometheus:** [http://localhost:9090](http://localhost:9090/) (Targets page should show `otel-collector:9464` as **UP**)
- **Loki (API, optional):** [http://localhost:3100/ready](http://localhost:3100/ready) → should return *ready*

When configuring Grafana data sources **in containers**, use the **service names**:

- Prometheus URL: `http://prometheus:9090`
- Loki URL: `http://loki:3100`
    
    (Using `localhost` inside Grafana’s container points to Grafana itself; use the container hostname per Docker Compose.) ([Grafana Labs](https://grafana.com/docs/learning-journeys/prometheus/add-data-source-url/?utm_source=chatgpt.com))
    

---

## 6) Import your dashboard (This is just shows the most important metrics,you can customize it as per your needs.)

In Grafana: **Dashboards → New → Import → Paste JSON** → Select your Prometheus/Loki data sources. ([Grafana Labs](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/import-dashboards/?utm_source=chatgpt.com))

**Dashboard JSON (paste exactly):**

```json
{  "annotations": {    "list": [      {        "builtIn": 1,        "datasource": {          "type": "grafana",          "uid": "-- Grafana --"        },        "enable": true,        "hide": true,        "iconColor": "rgba(0, 211, 255, 1)",        "name": "Annotations & Alerts",        "type": "dashboard"      }    ]  },  "editable": true,  "fiscalYearStartMonth": 0,  "graphTooltip": 0,  "id": 0,  "links": [],  "panels": [    {      "datasource": {        "type": "prometheus",        "uid": "ff16timaku1oga"      },      "fieldConfig": {        "defaults": {          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          }        },        "overrides": []      },      "gridPos": {        "h": 8,        "w": 24,        "x": 0,        "y": 0      },      "id": 4,      "options": {        "displayMode": "lcd",        "legend": {          "calcs": [],          "displayMode": "list",          "placement": "bottom",          "showLegend": false        },        "maxVizHeight": 300,        "minVizHeight": 16,        "minVizWidth": 8,        "namePlacement": "auto",        "orientation": "horizontal",        "reduceOptions": {          "calcs": [            "lastNotNull"          ],          "fields": "",          "values": false        },        "showUnfilled": true,        "sizing": "auto",        "valueMode": "color"      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "prometheus",            "uid": "ff16timaku1oga"          },          "expr": "sum by (gen_ai_token_type, gen_ai_request_model) (increase(gen_ai_client_token_usage_sum{developer_name=~\"$developer\", project_name=~\"$project\"}[1h]))",          "legendFormat": "{{gen_ai_request_model}} / {{gen_ai_token_type}}",          "refId": "A"        }      ],      "title": "Tokens used in last 1h",      "type": "bargauge"    },    {      "datasource": {        "type": "loki",        "uid": "ef17l48tlwcg0e"      },      "fieldConfig": {        "defaults": {          "color": {            "mode": "thresholds"          },          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          }        },        "overrides": []      },      "gridPos": {        "h": 10,        "w": 24,        "x": 0,        "y": 8      },      "id": 17,      "options": {        "colorMode": "value",        "graphMode": "area",        "justifyMode": "auto",        "orientation": "auto",        "percentChangeColorMode": "standard",        "reduceOptions": {          "calcs": [            "lastNotNull"          ],          "fields": "",          "values": false        },        "showPercentChange": false,        "textMode": "auto",        "wideLayout": true      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "loki",            "uid": "ef17l48tlwcg0e"          },          "direction": "backward",          "editorMode": "code",          "expr": "topk(10, sum by (programming_language) (count_over_time({service_name=~\"${service_name}\"} | event_name=\"gemini_cli.file_operation\" | session_id=~\"${session}\" | developer_name=~\"${developer}\" | project_name=~\"${project}\" | json | programming_language != \"\" [24h])))",          "legendFormat": "{{programming_language}}",          "queryType": "range",          "refId": "A"        }      ],      "title": "Top languages the user have interacted with (24h)",      "transparent": true,      "type": "stat"    },    {      "datasource": {        "type": "prometheus",        "uid": "ff16timaku1oga"      },      "fieldConfig": {        "defaults": {          "custom": {            "align": "auto",            "cellOptions": {              "type": "auto"            },            "footer": {              "reducers": []            },            "inspect": false          },          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          }        },        "overrides": []      },      "gridPos": {        "h": 8,        "w": 12,        "x": 0,        "y": 18      },      "id": 20,      "options": {        "cellHeight": "sm",        "showHeader": true,        "sortBy": [          {            "desc": true,            "displayName": "extension"          }        ]      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "prometheus",            "uid": "ff16timaku1oga"          },          "expr": "topk(10, sum by (extension) (increase(gemini_cli_file_operation_count_total{developer_name=~\"$developer\",project_name=~\"$project\"}[24h])))",          "legendFormat": "{{extension}}",          "refId": "A"        }      ],      "title": "Top extensions by file operations. (24h)",      "transformations": [        {          "id": "reduce",          "options": {            "labelsToFields": true,            "reducers": [              "lastNotNull"            ]          }        }      ],      "type": "table"    },    {      "datasource": {        "type": "prometheus",        "uid": "ff16timaku1oga"      },      "fieldConfig": {        "defaults": {          "custom": {            "align": "auto",            "cellOptions": {              "type": "auto"            },            "footer": {              "reducers": []            },            "inspect": false          },          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          }        },        "overrides": []      },      "gridPos": {        "h": 8,        "w": 12,        "x": 12,        "y": 18      },      "id": 21,      "options": {        "cellHeight": "sm",        "showHeader": true,        "sortBy": [          {            "desc": true,            "displayName": "mimetype"          }        ]      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "prometheus",            "uid": "ff16timaku1oga"          },          "expr": "topk(10, sum by (mimetype) (increase(gemini_cli_file_operation_count_total{developer_name=~\"$developer\",project_name=~\"$project\"}[24h])))",          "legendFormat": "{{mimetype}}",          "refId": "A"        }      ],      "title": "Top mimetypes by file operations (24h)",      "transformations": [        {          "id": "reduce",          "options": {            "labelsToFields": true,            "reducers": [              "lastNotNull"            ]          }        }      ],      "type": "table"    },    {      "datasource": {        "type": "loki",        "uid": "ef17l48tlwcg0e"      },      "fieldConfig": {        "defaults": {},        "overrides": []      },      "gridPos": {        "h": 7,        "w": 24,        "x": 0,        "y": 26      },      "id": 15,      "options": {        "dedupStrategy": "none",        "enableInfiniteScrolling": false,        "enableLogDetails": true,        "prettifyLogMessage": false,        "showCommonLabels": false,        "showLabels": false,        "showTime": true,        "sortOrder": "Descending",        "wrapLogMessage": true      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "loki",            "uid": "ef17l48tlwcg0e"          },          "direction": "backward",          "editorMode": "code",          "expr": "{service_name=~\"${service_name}\"} | event_name=\"gemini_cli.user_prompt\" | session_id=~\"${session}\" | developer_name=~\"${developer}\" | project_name=~\"${project}\" | logfmt | line_format \"{{ .prompt }}\"",          "queryType": "range",          "refId": "A"        }      ],      "title": "User prompts",      "type": "logs"    },    {      "datasource": {        "type": "loki",        "uid": "ef17l48tlwcg0e"      },      "fieldConfig": {        "defaults": {},        "overrides": []      },      "gridPos": {        "h": 10,        "w": 24,        "x": 0,        "y": 33      },      "id": 9,      "options": {        "dedupStrategy": "none",        "enableInfiniteScrolling": false,        "enableLogDetails": true,        "prettifyLogMessage": false,        "showCommonLabels": false,        "showLabels": false,        "showTime": true,        "sortOrder": "Descending",        "wrapLogMessage": true      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "loki",            "uid": "ef17l48tlwcg0e"          },          "direction": "backward",          "editorMode": "code",          "expr": "{service_name=~\"${service_name}\"} | event_name=~\"gemini_cli.*\" | session_id=~\"${session}\" | developer_name=~\"${developer}\" | project_name=~\"${project}\"",          "queryType": "range",          "refId": "A"        }      ],      "title": "Gemini events  — Filter by session/dev/project",      "type": "logs"    },    {      "datasource": {        "type": "prometheus",        "uid": "ff16timaku1oga"      },      "fieldConfig": {        "defaults": {          "color": {            "mode": "palette-classic"          },          "custom": {            "axisBorderShow": false,            "axisCenteredZero": false,            "axisColorMode": "text",            "axisLabel": "",            "axisPlacement": "auto",            "barAlignment": 0,            "barWidthFactor": 0.6,            "drawStyle": "line",            "fillOpacity": 0,            "gradientMode": "none",            "hideFrom": {              "legend": false,              "tooltip": false,              "viz": false            },            "insertNulls": false,            "lineInterpolation": "linear",            "lineWidth": 1,            "pointSize": 5,            "scaleDistribution": {              "type": "linear"            },            "showPoints": "auto",            "showValues": false,            "spanNulls": false,            "stacking": {              "group": "A",              "mode": "none"            },            "thresholdsStyle": {              "mode": "off"            }          },          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          },          "unit": "ops"        },        "overrides": []      },      "gridPos": {        "h": 14,        "w": 24,        "x": 0,        "y": 43      },      "id": 1,      "options": {        "legend": {          "calcs": [],          "displayMode": "list",          "placement": "bottom",          "showLegend": true        },        "tooltip": {          "hideZeros": false,          "mode": "single",          "sort": "none"        }      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "prometheus",            "uid": "ff16timaku1oga"          },          "editorMode": "code",          "expr": "sum by (model) (rate(gemini_cli_api_request_count_total{model=~\"$model\",developer_name=~\"$developer\",project_name=~\"$project\"}[5m]))",          "legendFormat": "{{model}}",          "range": true,          "refId": "A"        }      ],      "title": "Requests per second by model",      "type": "timeseries"    },    {      "datasource": {        "type": "prometheus",        "uid": "ff16timaku1oga"      },      "fieldConfig": {        "defaults": {          "color": {            "mode": "palette-classic"          },          "custom": {            "axisBorderShow": false,            "axisCenteredZero": false,            "axisColorMode": "text",            "axisLabel": "",            "axisPlacement": "auto",            "barAlignment": 0,            "barWidthFactor": 0.6,            "drawStyle": "line",            "fillOpacity": 0,            "gradientMode": "none",            "hideFrom": {              "legend": false,              "tooltip": false,              "viz": false            },            "insertNulls": false,            "lineInterpolation": "linear",            "lineWidth": 1,            "pointSize": 5,            "scaleDistribution": {              "type": "linear"            },            "showPoints": "auto",            "showValues": false,            "spanNulls": false,            "stacking": {              "group": "A",              "mode": "none"            },            "thresholdsStyle": {              "mode": "off"            }          },          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          },          "unit": "ops"        },        "overrides": []      },      "gridPos": {        "h": 10,        "w": 24,        "x": 0,        "y": 57      },      "id": 18,      "options": {        "legend": {          "calcs": [],          "displayMode": "list",          "placement": "bottom",          "showLegend": true        },        "tooltip": {          "hideZeros": false,          "mode": "single",          "sort": "none"        }      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "prometheus",            "uid": "ff16timaku1oga"          },          "expr": "sum by (operation) (rate(gemini_cli_file_operation_count_total{developer_name=~\"$developer\",project_name=~\"$project\"}[5m]))",          "legendFormat": "{{operation}}",          "refId": "A"        }      ],      "title": "File operations per second by type (metrics)",      "type": "timeseries"    },    {      "datasource": {        "type": "prometheus",        "uid": "ff16timaku1oga"      },      "fieldConfig": {        "defaults": {          "color": {            "mode": "palette-classic"          },          "custom": {            "axisBorderShow": false,            "axisCenteredZero": false,            "axisColorMode": "text",            "axisLabel": "",            "axisPlacement": "auto",            "barAlignment": 0,            "barWidthFactor": 0.6,            "drawStyle": "line",            "fillOpacity": 0,            "gradientMode": "none",            "hideFrom": {              "legend": false,              "tooltip": false,              "viz": false            },            "insertNulls": false,            "lineInterpolation": "linear",            "lineWidth": 1,            "pointSize": 5,            "scaleDistribution": {              "type": "linear"            },            "showPoints": "auto",            "showValues": false,            "spanNulls": false,            "stacking": {              "group": "A",              "mode": "none"            },            "thresholdsStyle": {              "mode": "off"            }          },          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          }        },        "overrides": []      },      "gridPos": {        "h": 10,        "w": 24,        "x": 0,        "y": 67      },      "id": 5,      "options": {        "legend": {          "calcs": [],          "displayMode": "list",          "placement": "bottom",          "showLegend": true        },        "tooltip": {          "hideZeros": false,          "mode": "single",          "sort": "none"        }      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "prometheus",            "uid": "ff16timaku1oga"          },          "expr": "sum by (function_name) (rate(gemini_cli_tool_call_count_total{developer_name=~\"$developer\",project_name=~\"$project\"}[5m]))",          "legendFormat": "{{function_name}}",          "refId": "A"        }      ],      "title": "Tool calls per second by function",      "type": "timeseries"    },    {      "datasource": {        "type": "prometheus",        "uid": "ff16timaku1oga"      },      "fieldConfig": {        "defaults": {          "custom": {            "align": "auto",            "cellOptions": {              "type": "auto"            },            "footer": {              "reducers": []            },            "inspect": false          },          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          }        },        "overrides": []      },      "gridPos": {        "h": 8,        "w": 24,        "x": 0,        "y": 77      },      "id": 8,      "options": {        "cellHeight": "sm",        "showHeader": true      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "prometheus",            "uid": "ff16timaku1oga"          },          "expr": "topk(10, sum by (gen_ai_request_model) (increase(gen_ai_client_token_usage_sum{gen_ai_token_type=\"output\",developer_name=~\"$developer\",project_name=~\"$project\"}[24h])))",          "legendFormat": "{{gen_ai_request_model}}",          "refId": "A"        }      ],      "title": "Top models by output tokens (24h)",      "transformations": [        {          "id": "reduce",          "options": {            "labelsToFields": true,            "reducers": [              "lastNotNull"            ]          }        }      ],      "type": "table"    },    {      "datasource": {        "type": "loki",        "uid": "ef17l48tlwcg0e"      },      "fieldConfig": {        "defaults": {          "color": {            "mode": "thresholds"          },          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          },          "unit": "chars"        },        "overrides": []      },      "gridPos": {        "h": 9,        "w": 12,        "x": 0,        "y": 85      },      "id": 14,      "options": {        "minVizHeight": 75,        "minVizWidth": 75,        "orientation": "auto",        "reduceOptions": {          "calcs": [            "lastNotNull"          ],          "fields": "",          "values": false        },        "showThresholdLabels": false,        "showThresholdMarkers": true,        "sizing": "auto"      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "loki",            "uid": "ef17l48tlwcg0e"          },          "direction": "backward",          "editorMode": "code",          "expr": "avg by () (avg_over_time(({service_name=~\"${service_name}\"} | event_name=\"gemini_cli.user_prompt\" | session_id=~\"${session}\" | developer_name=~\"${developer}\" | project_name=~\"${project}\" | logfmt | unwrap prompt_length | __error__=\"\")[5m]))",          "queryType": "range",          "refId": "A"        }      ],      "title": "Avg user prompt length (5m) — logs",      "type": "gauge"    },    {      "datasource": {        "type": "prometheus",        "uid": "ff16timaku1oga"      },      "fieldConfig": {        "defaults": {          "color": {            "mode": "palette-classic"          },          "custom": {            "axisBorderShow": false,            "axisCenteredZero": false,            "axisColorMode": "text",            "axisLabel": "",            "axisPlacement": "auto",            "barAlignment": 0,            "barWidthFactor": 0.6,            "drawStyle": "line",            "fillOpacity": 0,            "gradientMode": "none",            "hideFrom": {              "legend": false,              "tooltip": false,              "viz": false            },            "insertNulls": false,            "lineInterpolation": "linear",            "lineWidth": 1,            "pointSize": 5,            "scaleDistribution": {              "type": "linear"            },            "showPoints": "auto",            "showValues": false,            "spanNulls": false,            "stacking": {              "group": "A",              "mode": "none"            },            "thresholdsStyle": {              "mode": "off"            }          },          "mappings": [],          "thresholds": {            "mode": "absolute",            "steps": [              {                "color": "green",                "value": 0              },              {                "color": "red",                "value": 80              }            ]          },          "unit": "ops"        },        "overrides": []      },      "gridPos": {        "h": 9,        "w": 12,        "x": 12,        "y": 85      },      "id": 2,      "options": {        "legend": {          "calcs": [],          "displayMode": "list",          "placement": "bottom",          "showLegend": true        },        "tooltip": {          "hideZeros": false,          "mode": "single",          "sort": "none"        }      },      "pluginVersion": "12.2.0",      "targets": [        {          "datasource": {            "type": "prometheus",            "uid": "ff16timaku1oga"          },          "editorMode": "code",          "expr": "sum by (status_code) (rate(gemini_cli_api_request_count_total{status_code=~\"4..|5..\",model=~\"$model\",developer_name=~\"$developer\",project_name=~\"$project\"}[5m]))",          "legendFormat": "{{status_code}}",          "refId": "A"        }      ],      "title": "Error rate by HTTP status (4xx/5xx)",      "type": "timeseries"    }  ],  "preload": false,  "refresh": "5s",  "schemaVersion": 42,  "tags": [    "gemini",    "otel",    "prometheus",    "loki"  ],  "templating": {    "list": [      {        "allValue": ".*",        "current": {          "text": "All",          "value": "$__all"        },        "datasource": {          "type": "prometheus",          "uid": "ff16timaku1oga"        },        "includeAll": true,        "multi": true,        "name": "model",        "options": [],        "query": "label_values(gemini_cli_api_request_count_total, model)",        "refresh": 1,        "type": "query"      },      {        "allValue": ".+",        "current": {          "text": "All",          "value": "$__all"        },        "datasource": {          "type": "loki",          "uid": "ef17l48tlwcg0e"        },        "description": "Pick service (OTLP → Loki index label). All uses .+ to satisfy LogQL.",        "includeAll": true,        "multi": true,        "name": "service_name",        "options": [],        "query": "label_values({service_name=~\".+\"}, service_name)",        "refresh": 1,        "type": "query"      },      {        "current": {          "text": ".*",          "value": ".*"        },        "description": "Regex for session_id (structured metadata)",        "name": "session",        "options": [          {            "selected": true,            "text": ".*",            "value": ".*"          }        ],        "query": ".*",        "type": "textbox"      },      {        "allValue": ".*",        "current": {          "text": [            "Damian Peiris"          ],          "value": [            "Damian Peiris"          ]        },        "datasource": {          "type": "prometheus",          "uid": "ff16timaku1oga"        },        "includeAll": true,        "multi": true,        "name": "developer",        "options": [],        "query": "label_values(gemini_cli_api_request_count_total, developer_name)",        "refresh": 1,        "type": "query"      },      {        "allValue": ".*",        "current": {          "text": [            "Gemini Observability"          ],          "value": [            "Gemini Observability"          ]        },        "datasource": {          "type": "prometheus",          "uid": "ff16timaku1oga"        },        "includeAll": true,        "multi": true,        "name": "project",        "options": [],        "query": "label_values(gemini_cli_api_request_count_total, project_name)",        "refresh": 1,        "type": "query"      }    ]  },  "time": {    "from": "now-24h",    "to": "now"  },  "timepicker": {},  "timezone": "",  "title": "Gemini CLI — Metrics & Logs",  "uid": "gemini-cli-all",  "version": 50}
```

---

## 7) What the panels calculate (plain English)

- **Tokens used in last 1h (GenAI metric)**
    
    `increase(gen_ai_client_token_usage_sum[1h])` grouped by `gen_ai_token_type` and `gen_ai_request_model`. Shows token deltas in the last hour.
    
- **Requests per second by model**
    
    `rate(gemini_cli_api_request_count_total[5m])` → 5-min moving RPS per model.
    
- **Error rate by status (4xx/5xx)**
    
    Filters status codes with `status_code=~"4..|5.."` then `rate(...)` by status.
    
- **Tool calls per second by function**
    
    `rate(gemini_cli_tool_call_count_total[5m])` grouped by `function_name`.
    
- **Top models by output tokens (24h)**
    
    `topk(10, sum by (gen_ai_request_model) (increase(gen_ai_client_token_usage_sum{gen_ai_token_type="output"}[24h])))`.
    
- **Tool call success ratio (5m)**
    
    `sum(rate(...{success="true"}[5m])) / sum(rate(...[5m]))`.
    
- **Sessions started (24h)**
    
    `increase(gemini_cli_session_count_total[24h])`.
    
- **Logs panels (Loki)**
    
    LogQL queries begin with a **stream selector** like `{service_name=~"..."};` filters then compute rates/quantiles using pipeline stages (e.g., `| unwrap duration_ms`). LogQL requires at least one label matcher in the selector. ([Grafana Labs](https://grafana.com/docs/loki/latest/query/log_queries/?utm_source=chatgpt.com))
    

---

## 8) Verify data quickly

- **Prometheus target:** `http://localhost:9090/targets` → `otel-collector:9464` **UP**
- **Collector metrics endpoint (optional):** `curl http://localhost:9464/metrics` (you’ll see Prom metrics; default exporter port is 9464). ([OpenTelemetry](https://opentelemetry.io/docs/specs/otel/metrics/sdk_exporters/prometheus/?utm_source=chatgpt.com))
- **Loki readiness:** `curl http://localhost:3100/ready` → “ready”
- **Grafana data sources:** Use **Prometheus** URL `http://prometheus:9090` and **Loki** URL `http://loki:3100`. ([Grafana Labs](https://grafana.com/docs/learning-journeys/prometheus/add-data-source-url/?utm_source=chatgpt.com))

---

## 9) Troubleshooting (fast)

- **Grafana can’t reach Prometheus/Loki:** Inside containers, **don’t use `localhost`**; use service names (`prometheus`, `loki`). ([Grafana Labs](https://grafana.com/docs/learning-journeys/prometheus/add-data-source-url/?utm_source=chatgpt.com))
- **Loki queries error “need a stream selector”:** Start queries with a selector like `{service_name=~".+"}`, then pipe filters. ([Grafana Labs](https://grafana.com/docs/loki/latest/query/log_queries/?utm_source=chatgpt.com))
- **No metrics in panels yet:** Drive Gemini CLI a bit; metrics like `gemini_cli_*` and `gen_ai_*` appear only after events.

---
