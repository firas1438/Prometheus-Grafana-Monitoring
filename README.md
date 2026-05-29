# Prometheus & Grafana Monitoring

A simple Docker setup to monitor host system metrics (CPU, Memory, Disk, Network) using Prometheus, Node Exporter, and Grafana.

## Setup & Usage

### I. Start the services

```bash
docker compose up -d
```

### II. Access the services

| Service | URL |
|---|---|
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 |
| Node Exporter | http://node-exporter:9100 |

> Node Exporter uses `expose` instead of `ports`, it's intentionally unreachable via browser because only Prometheus should access it.

### III. Configure Grafana data source

1. Go to **Connections → Data sources** and add Prometheus.
2. Set the URL to `http://prometheus:9090`.
3. Click **Save & Test** - the connection should succeed.

### IV. Set up a dashboard

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


## Flow Summary

```
Host Machine
    ↓
Node Exporter
    ↓
Prometheus
    ↓
Grafana
```

Prometheus scrapes metrics from Node Exporter every 15 seconds and stores them in its time-series database. Grafana queries and visualizes that data.


## Notes

> Prometheus scrapes metrics using Docker service names within the `monitoring` network, not `localhost`.