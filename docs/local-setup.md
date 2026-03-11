# Local Infrastructure Setup

Local pipeline runs JMeter tests through Jenkins CI with real-time metrics streaming to InfluxDB and Grafana.

## Components

| Tool | Version | Purpose |
|------|---------|---------|
| Apache JMeter | 5.5 | Test execution |
| InfluxDB | 1.x | Metrics storage |
| Grafana | 12.x | Dashboards |
| Telegraf | 1.x | System metrics agent |
| Jenkins | LTS | CI orchestration |

> InfluxDB 1.x is required. Version 2.x uses Flux — incompatible with JMeter Backend Listener and standard Grafana dashboards.

---

## How it works

Jenkins clones this repository via SSH, executes `ui_ecommerce_flow.jmx` with build parameters, and archives results. During the test, JMeter streams metrics to InfluxDB via Backend Listener. Grafana reads from InfluxDB and displays live dashboards. Telegraf independently collects system metrics into a separate `telegraf` database.

```
Jenkins ──► JMeter ──► InfluxDB (db: jmeter_shopizer) ──► Grafana
Telegraf ──────────► InfluxDB (db: telegraf)          ──► Grafana
```

---

## Directory Layout

Tools are stored outside the Jenkins workspace to avoid being wiped between builds. Jenkins build commands use absolute paths.

```
~/Tools/
  ├── apache-jmeter-5.5/
  ├── influxdb-1.x/
  ├── grafana-12.x/
  └── telegraf/
```

---

## JMeter Plugins

This test plan requires non-standard plugins. Install via Plugin Manager GUI or manually:

| Plugin ID | Purpose |
|-----------|---------|
| `jpgc-prmctl` | `SetVariablesAction` — set variables between transactions |
| `jpgc-csl` | `ConsoleStatusLogger` — live stats in non-GUI mode |

`WeightedSwitchController` is not available in the Plugin Manager repository. Install manually by copying `jmeter-plugins-wsc-0.7.jar` to `~/Tools/apache-jmeter-5.5/lib/ext/`. The jar is bundled in the [`/lib`](../lib/) folder of this repository.

---

## InfluxDB

Start: `./influxd` (port 8086)

Two databases are required:

```sql
CREATE DATABASE jmeter_shopizer  -- JMeter Backend Listener metrics
CREATE DATABASE telegraf          -- Telegraf system metrics
```

> Use the full path to the influx CLI if it is not in PATH: `~/Tools/influxdb-1.x/influx`

---

## Telegraf

Configuration (`telegraf.conf`):

```toml
[[outputs.influxdb]]
  urls     = ["http://127.0.0.1:8086"]
  database = "telegraf"

[[inputs.cpu]]
  percpu   = true
  totalcpu = true
[[inputs.mem]]
[[inputs.disk]]
[[inputs.net]]
[[inputs.system]]
```

Start: `./telegraf --config telegraf.conf`

---

## Grafana

Start: `./bin/grafana-server` (port 3000)

**Data sources:**

| Name | Query Language | URL | Database |
|------|---------------|-----|---------|
| jmeter_shopizer | InfluxQL | http://localhost:8086 | jmeter_shopizer |
| telegraf | InfluxQL | http://localhost:8086 | telegraf |

**Dashboards** — import by Grafana.com ID:

| Dashboard | ID | UID | Data Source |
|-----------|----|-----|------------|
| Load Test Monitoring (Shopizer) | 1152 | `shopizer-load-test` | jmeter_shopizer |
| Comparison Dashboard (Shopizer) | 5496 | `shopizer-comparison` | jmeter_shopizer |
| Linux System Overview | 928 | — | telegraf |

> When importing dashboards 1152 and 5496, change the UID to the values above to avoid conflicts if Project 1 dashboards are also installed on the same Grafana instance.

**Anonymous access** is enabled in `grafana.ini` so Jenkins build description links open dashboards directly without login:

```ini
[auth.anonymous]
enabled  = true
org_name = Main Org.
org_role = Viewer
```

---

## JMeter — Backend Listener

`ui_ecommerce_flow.jmx` contains a Backend Listener configured to write to InfluxDB:

| Parameter | Value |
|-----------|-------|
| Implementation | `InfluxdbBackendListenerClient` |
| influxdbUrl | `${__P(INFLUX_DB,http://localhost:8086/write?db=jmeter_shopizer)}` |
| application | `${__TestPlanName}@${__time(yyyy-MM-dd'T'hh:mm:ss,)}` |
| summaryOnly | `false` |
| percentiles | `50;90;95;99` |

`backend_metrics_window_mode=timed` is set in `jmeter.properties`. Without this, percentiles in Grafana diverge from the HTML Aggregate Report because the default `fixed` mode smooths metrics from test start rather than recalculating per time window.

---

## Test Data — users_creds.csv

`test-plans/users_creds.csv` contains pre-generated credentials for users registered on Shopizer. It is committed to the repository and used by the CSV Data Set Config in the main Thread Group for Login transactions.

**When `CREATE_USERS=FALSE`** (default) — the test reads credentials directly from the committed CSV.

**When `CREATE_USERS=TRUE`** — setUp Thread Group deletes the existing CSV, registers new users on Shopizer, and writes fresh credentials. This overwrites the file only within the Jenkins workspace — the repository is not modified automatically.

To refresh test data in the repository after a `CREATE_USERS=TRUE` run:

```bash
cp ~/.jenkins/workspace/Shopizer_UI_Ecommerce/test-plans/users_creds.csv \
   path/to/shopizer-ui-performance-tests/test-plans/users_creds.csv

git add test-plans/users_creds.csv
git commit -m "chore: refresh test user credentials"
git push origin main
```

---

## Jenkins

Start: `java -jar jenkins.war --httpPort=8080`

**Plugins:** Groovy Postbuild (Git is pre-installed)

**SSH credentials** — repository is cloned via SSH key stored in Jenkins credentials (`github-ssh`).

**One Freestyle job:** `Shopizer_UI_Ecommerce`

Job is parameterized with the following parameters:

| Name | Default | Description |
|------|---------|-------------|
| USERS | 1 | Number of virtual users |
| RAMPUP | 60 | Ramp-up period (seconds) |
| DURATION | 120 | Test duration (seconds) |
| LOOPS | -1 | -1 = duration-controlled |
| PRODUCTS_TO_ADD_COUNT | 2 | Products to add per iteration |
| CREATE_USERS | FALSE | Run setUp Thread Group (TRUE/FALSE) |
| USERS_CREATION_COUNT | 6 | Users to register in setUp |
| TEST_TITLE | — | Mandatory. Validating String with regex `.+` |

Build command:

```bash
/path/to/Tools/apache-jmeter-5.5/bin/jmeter.sh -n \
  -t test-plans/ui_ecommerce_flow.jmx \
  -l results.csv -j jmeter.log -e -o report -f \
  -JUSERS=$USERS -JRAMPUP=$RAMPUP -JDURATION=$DURATION -JLOOPS=$LOOPS \
  -JPRODUCTS_TO_ADD_COUNT=$PRODUCTS_TO_ADD_COUNT \
  -JCREATE_USERS=$CREATE_USERS \
  -JUSERS_CREATION_COUNT=$USERS_CREATION_COUNT \
  -JTEST_TITLE="$TEST_TITLE" \
  -Jbackend_metrics_window_mode=timed \
  -Jbackend_metrics_window=5000 \
  -JINFLUX_DB=http://localhost:8086/write?db=jmeter_shopizer
```

**Post-build:** Groovy Postbuild generates a Grafana link with the exact test time range in the build description:

```groovy
import hudson.model.*
def build      = Thread.currentThread().executable
def grafanaUrl = System.getenv("JENKINS_GRAFANA_URL") ?: "127.0.0.1:3000"
def start      = build.getStartTimeInMillis()
def end        = start + build.getExecutor().getElapsedTime()
def url = String.format(
    "http://%s/d/shopizer-load-test/load-test-monitoring-dashboard-shopizer?from=%s&to=%s",
    grafanaUrl, start, end
)
build.setDescription("<a href='${url}'>Grafana Performance Result</a>")
```

---
