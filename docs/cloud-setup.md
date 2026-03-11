# Cloud Infrastructure Setup

Cloud pipeline runs the JMeter UI ecommerce scenario automatically on every push to `main`, streaming metrics to a persistent GCP VM.

## Components

| Tool | Where | Purpose |
|------|-------|---------|
| GitHub Actions | GitHub-hosted runner | Test execution |
| GCP VM (e2-micro, us-central1) | Google Cloud | Hosts InfluxDB + Grafana + Telegraf |
| InfluxDB 1.x | GCP VM | Metrics storage (db: jmeter_shopizer_gha) |
| Grafana | GCP VM | Dashboards |
| Telegraf | GCP VM | VM system metrics |

---

## How it works

On push to `main` (or manual trigger via `workflow_dispatch`), GitHub Actions installs JMeter 5.5 with required plugins and runs `ui_ecommerce_flow.jmx`. JMeter streams metrics to InfluxDB on the GCP VM during the run. After the test completes, results are uploaded as GitHub Actions artifacts.

```
GitHub Actions ──► JMeter ──► GCP VM: InfluxDB (db: jmeter_shopizer_gha) ──► Grafana
                              GCP VM: Telegraf ──────────────────────────► Grafana
```

---

## Pipeline Isolation

Local and cloud pipelines write to separate InfluxDB databases. The `INFLUX_DB` parameter defaults to `localhost` for local runs and is overridden via GitHub Secret for cloud runs — same `.jmx` file works in both environments.

| Pipeline | Host | Database |
|----------|------|---------|
| Jenkins (local) | localhost:8086 | jmeter_shopizer |
| GitHub Actions | GCP VM:8086 | jmeter_shopizer_gha |

---

## GCP VM

**Specs:** e2-micro, us-central1, Ubuntu 22.04 LTS

**Firewall rules:**

| Port | Source | Purpose |
|------|--------|---------|
| 8086 | 0.0.0.0/0 | InfluxDB — GitHub Actions uses dynamic IPs |
| 3000 | your IP | Grafana — restrict access |

---

## InfluxDB on GCP VM

```bash
sudo apt install -y influxdb influxdb-client
sudo systemctl enable --now influxdb
influx -execute 'CREATE DATABASE jmeter_shopizer_gha'
```

---

## Grafana on GCP VM

Installed via apt. Data sources:

| Name | Query Language | URL | Database |
|------|---------------|-----|---------|
| jmeter_shopizer_gha | InfluxQL | http://localhost:8086 | jmeter_shopizer_gha |
| telegraf | InfluxQL | http://localhost:8086 | telegraf |

Import dashboards by Grafana.com ID:

| Dashboard | ID | UID | Data Source |
|-----------|----|-----|------------|
| Load Test Monitoring (Shopizer) | 1152 | `shopizer-gha` | jmeter_shopizer_gha |
| Comparison Dashboard (Shopizer) | 5496 | `shopizer-gha-cmp` | jmeter_shopizer_gha |
| Linux System Overview | 928 | — | telegraf |

---

## Telegraf on GCP VM

```toml
[[outputs.influxdb]]
  urls     = ["http://127.0.0.1:8086"]
  database = "telegraf"

[[inputs.cpu]]
[[inputs.mem]]
[[inputs.disk]]
[[inputs.net]]
[[inputs.system]]
[[inputs.processes]]
[[inputs.swap]]
```

---

## GitHub Actions Workflow

**Secret required:**

| Name | Value |
|------|-------|
| `INFLUX_WRITE_URL` | `http://<VM_IP>:8086/write?db=jmeter_shopizer_gha` |

> GCP VM IP changes on restart — update this secret accordingly.

**Workflow inputs (`workflow_dispatch`):**

| Input | Default | Description |
|-------|---------|-------------|
| `users` | 1 | Number of virtual users |
| `rampup` | 60 | Ramp-up period (seconds) |
| `duration` | 120 | Test duration (seconds) |
| `products_to_add_count` | 2 | Products to add per iteration |
| `users_creation_count` | 6 | Users to register in setUp |
| `create_users` | FALSE | Run setUp Thread Group (TRUE/FALSE) |
| `test_title` | GHA - UI Ecommerce Flow | Test name visible in Grafana |

**JMeter plugins** installed during the workflow:

| Method | Plugins |
|--------|---------|
| Plugin Manager | `jpgc-synthesis`, `jpgc-graphs-vs`, `jpgc-tst`, `jpgc-dummy`, `jpgc-prmctl`, `jpgc-csl` |
| Bundled jar (from `/lib`) | `jmeter-plugins-wsc-0.7.jar` |

> `WeightedSwitchController` is not available in the Plugin Manager repository and cannot be downloaded from Maven Central in CI. The jar is committed to the repository under `/lib` and copied to `lib/ext` during the workflow.

**Key JMeter flags passed in the workflow:**

```yaml
-JINFLUX_DB=${{ secrets.INFLUX_WRITE_URL }}
-JCREATE_USERS=${{ env.CREATE_USERS }}
-Jbackend_metrics_window_mode=timed
-Jbackend_metrics_window=5000
```

**Artifacts** uploaded after run (even on failure via `if: always()`):
- `results-ui-ecommerce.csv`
- `jmeter-ui-ecommerce.log`
- `html-report-ui-ecommerce/`

---

## Test Data in Cloud Pipeline

`test-plans/users_creds.csv` is committed to the repository and available on the GitHub Actions runner after checkout.

**When `create_users=FALSE`** (default) — JMeter reads credentials from the committed CSV. No setUp registration happens, test starts immediately.

**When `create_users=TRUE`** — setUp Thread Group registers new users on Shopizer before the load test. The generated CSV exists only within the runner for the duration of the run and is not persisted back to the repository. Download the updated CSV from workflow artifacts to refresh credentials in the repository.

---
