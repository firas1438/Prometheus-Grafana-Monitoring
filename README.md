## Prometheus & Grafana Monitoring

A simple Docker setup to monitor host system metrics (CPU, Memory, Disk, Network) using Prometheus, Node Exporter, and Grafana, with alerting to Slack through Alertmanager.

## Setup & Usage

### I. Create the Slack webhook secret

Alertmanager sends alerts to Slack through an [Incoming Webhook](https://api.slack.com/messaging/webhooks). The URL is a secret, so it is stored in a file that is ignored by Git.

1. Create a Slack app with **Incoming Webhooks** enabled and copy the webhook URL.
2. Copy the example file:

```bash
cp alertmanager/slack_webhook_url.example alertmanager/slack_webhook_url
```

3. Open `alertmanager/slack_webhook_url` and replace the placeholder with your real URL (one line, nothing else).

> This file must exist **before** starting the stack, otherwise Docker Compose fails with an error. It is listed in `.gitignore`, so it is never committed.

### II. Start the services

```bash
docker compose up -d
```

### III. Access the services

| Service      | URL                   |
| ------------ | --------------------- |
| Prometheus   | http://localhost:9090 |
| Grafana      | http://localhost:3000 |
| Alertmanager | http://localhost:9093 |

> Node Exporter (`node-exporter:9100`) is internal only. It uses `expose` instead of `ports`, so it is intentionally unreachable from the browser. Only Prometheus should access it.

### IV. Configure Grafana data source

1. Go to **Connections → Data sources** and add Prometheus.
2. Set the URL to `http://prometheus:9090`.
3. Click **Save & Test** - the connection should succeed.

### V. Set up a dashboard

#### Option A - Import a pre-built dashboard

1. Go to **Dashboards → Import**.
2. Enter dashboard ID `1860`.
3. Select Prometheus as the data source and click **Import**.

![Pre-built dashboard](https://i.imgur.com/AIRhecE.png)

#### Option B - Build a custom dashboard

1. Create a JSON file at `grafana/dashboards/dashboard.json`.
2. Define panels, layout, and PromQL queries inside it.
3. Grafana will automatically load it via dashboard provisioning.

![Custom dashboard](https://i.imgur.com/6UVxKmK.png)

## Alerting

Prometheus evaluates the rules in `prometheus/alert.rules.yml` every 15 seconds. When a rule stays true for its `for` duration, Prometheus sends the alert to Alertmanager, which forwards it to Slack (configured in `alertmanager/alertmanager.yml`).

| Alert             | Condition                     | For | Severity |
| ----------------- | ----------------------------- | --- | -------- |
| `HostDown`        | Node Exporter is unreachable  | 1m  | critical |
| `HighCpuUsage`    | CPU usage above 80%           | 2m  | warning  |
| `HighMemoryUsage` | Memory usage above 85%        | 2m  | warning  |
| `DiskAlmostFull`  | Less than 15% disk space left | 5m  | warning  |

Alertmanager also sends a `[RESOLVED]` message when the problem is fixed.

### I. Testing the alerts

Generate load inside the Docker VM so Node Exporter sees it. Run one test at a time and wait for the `[RESOLVED]` message before starting the next.

**HostDown**

```bash
docker compose stop node-exporter
# wait ~1 minute for the alert, then:
docker compose start node-exporter
```

**HighCpuUsage**

```bash
docker run --rm alpine sh -c "apk add --no-cache stress-ng && stress-ng --cpu 0 --timeout 240s"
```

**HighMemoryUsage**

```bash
docker run --rm alpine sh -c "apk add --no-cache stress-ng && stress-ng --vm 1 --vm-bytes 90% --timeout 240s"
```

**DiskAlmostFull**

A disk is hard to fill on purpose, so temporarily change `< 15` to `< 99` in `prometheus/alert.rules.yml`, run `docker compose restart prometheus`, then revert the change.

### II. Result

Slack notifications received during the tests (`HostDown` and `HighCpuUsage`), next to the Grafana dashboard showing the CPU spike:

![Slack alert notifications](https://i.imgur.com/MrqljTl.png)

## Flow Summary

![Flow diagram](https://i.imgur.com/1WkbSKy.png)

Prometheus scrapes metrics from Node Exporter every 15 seconds and stores them in its time-series database. Grafana queries and visualizes that data. Prometheus also evaluates the alert rules and sends firing alerts to Alertmanager, which notifies Slack.

## Notes

> On Windows/macOS (Docker Desktop), the metrics reflect the Docker Linux VM, not the host OS. On a Linux host or an EC2 instance, they reflect the real machine.

> After editing `prometheus.yml`, `alert.rules.yml`, or `alertmanager.yml`, restart the affected service (for example `docker compose restart prometheus`), since they only read their config at startup.
