# Shopizer UI Performance Tests

Performance testing suite for [Shopizer](https://github.com/shopizer-ecommerce/shopizer) e-commerce application built with Apache JMeter. Full CI/CD pipeline with real-time metrics visualization: locally via Jenkins and in the cloud via GitHub Actions + Google Cloud Platform.

---

## Stack

| Tool | Role |
|------|------|
| Apache JMeter 5.5 | Load test execution, metrics via Backend Listener |
| InfluxDB 1.x | Time-series storage for JMeter and system metrics |
| Grafana | Real-time dashboards: Load Test + Comparison + Server Monitoring |
| Telegraf | System-level metrics agent (CPU/RAM/Disk/Network) |
| Jenkins | Local CI: parameterized builds, HTML reports, Grafana links |
| GitHub Actions | Cloud CI: automated test runs on push/manual trigger |
| Google Cloud Platform | Hosts InfluxDB + Grafana + Telegraf for cloud pipeline |

---

## Architecture

**Local Pipeline**
```
Jenkins
  └── JMeter ──► InfluxDB (db: jmeter_shopizer) ──► Grafana
Telegraf ────► InfluxDB (db: telegraf) ──────────► Grafana
```

**Cloud Pipeline**
```
git push/workflow_dispatch
  └── GitHub Actions
        └── JMeter ──► GCP VM: InfluxDB (db: jmeter_shopizer_gha) ──► Grafana
              GCP VM: Telegraf ──────────────────────────────────────► Grafana
```

> Local and cloud pipelines are fully isolated.
> Same `.jmx` file, two environments, via `INFLUX_DB` parameter with default fallback.

---

## Test Scenario

Full UI e-commerce journey: open home page - login - browse category - open product - add to cart - checkout - submit order.

All parameters are externalized via `${__P(VARIABLE, default)}` - no hardcoded values.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `USERS` | 1 | Number of virtual users |
| `RAMPUP` | 60 | Ramp-up period (seconds) |
| `DURATION` | 120 | Test duration (seconds) |
| `LOOPS` | -1 | -1 = duration-controlled |
| `PRODUCTS_TO_ADD_COUNT` | 2 | Products to add per session |
| `CREATE_USERS` | FALSE | Run setUp registration (TRUE/FALSE) |
| `USERS_CREATION_COUNT` | 6 | Users to register in setUp |
| `TEST_TITLE` | local_test_run | Test name visible in Grafana |

See [scenario/scenario.md](scenario/scenario.md) for full scenario description and JMX screenshots.

---

## Performance Results

**Load test:** 5 users, ramp-up 60s, duration 3600s - stable run, 0% errors, avg response time 176ms.

See [PERFORMANCE_RESULTS.md](PERFORMANCE_RESULTS.md) for full aggregate report.

---

## Grafana Dashboards

Real-time metrics visualized during test execution across 3 dashboards.

| Dashboard | Purpose | Data Source |
|-----------|---------|-------------|
| JMeter Load Test Monitoring | Active threads, RPS, response time, error rate | InfluxDB `jmeter_shopizer` |
| JMeter Comparison Dashboard | Side-by-side comparison across test runs | InfluxDB `jmeter_shopizer` |
| Linux System Overview | CPU/RAM/Disk/Network during test | InfluxDB `telegraf` |

Dashboard screenshots from live test runs:

| Screenshot | Description |
|-----------|-------------|
| [Load Test Monitoring (5u 300s)](dashboards/screenshots/load-test-monitoring-5u-300s.png) | Active threads, RPS, response time |
| [Load Test Monitoring (15u 1800s)](dashboards/screenshots/load-test-monitoring-15u-1800s.png) | Higher load profile |
| [Comparison Report](dashboards/screenshots/comparison-report.png) | Two runs side-by-side |
| [Linux System Overview](dashboards/screenshots/linux-system-overview.png) | CPU/RAM during test |

> Dashboards are not exported as JSON - they contain local data source references. To reproduce, import community dashboards by ID (1152, 5496, 928) and configure data sources per [docs/local-setup.md](docs/local-setup.md).

---

## CI/CD

### GitHub Actions (Cloud)
Runs the scenario on every push to `main` or manually via `workflow_dispatch`. Results uploaded as artifacts (CSV, HTML report, log).
-> [View workflow runs](../../actions/workflows/jmeter.yml)

### Jenkins (Local)
Parameterized freestyle job. Each build archives `results.csv`, `jmeter.log`, HTML report and generates a Grafana link with exact test time range in the build description.

| Screenshot | Description |
|-----------|-------------|
| [Build History](docs/screenshots/jenkins/jenkins-build-history.png) | Shopizer_UI_Ecommerce build history |
| [Build Parameters](docs/screenshots/jenkins/jenkins-build-parameters.png) | Parameterized build form |
| [Console Output](docs/screenshots/jenkins/jenkins-console-output.png) | JMeter execution with parameters |

---

## Quick Start

### Local (Jenkins)
-> [docs/local-setup.md](docs/local-setup.md)

### Cloud (GitHub Actions + GCP)
-> [docs/cloud-setup.md](docs/cloud-setup.md)

### Manual CLI run
```bash
/path/to/jmeter.sh -n \
  -t test-plans/ui_ecommerce_flow.jmx \
  -l results.csv -e -o report \
  -JUSERS=5 -JRAMPUP=60 -JDURATION=300 \
  -JTEST_TITLE="smoke_test"
```

---

## Repository Structure

```
shopizer-ui-performance-tests/
├── .github/
│   └── workflows/
│       └── jmeter.yml                  # Cloud CI pipeline (GitHub Actions)
├── test-plans/
│   ├── ui_ecommerce_flow.jmx           # Full UI e-commerce flow
│   └── users_creds.csv                 # Pre-generated test credentials
├── dashboards/
│   ├── screenshots/                    # Grafana dashboard screenshots
│   └── README.md                       # Dashboard overview
├── docs/
│   ├── screenshots/
│   │   └── jenkins/                    # Jenkins CI screenshots
│   ├── local-setup.md                  # Local pipeline setup
│   └── cloud-setup.md                  # Cloud pipeline setup
├── lib/
│   └── jmeter-plugins-wsc-0.7.jar      # WeightedSwitchController plugin
├── scenario/
│   ├── screenshots/                    # JMX screenshots
│   └── scenario.md                     # Test scenario description
├── .gitignore
├── PERFORMANCE_RESULTS.md              # Test results summary
└── README.md
```

---

