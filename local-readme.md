# Grafana and Loki Log Collection Setup

This guide explains how to set up **Grafana** and **Loki** using Docker Compose to collect and visualize logs from another server running Docker containers.

---

## Prerequisites

Before you begin, ensure you have the following:

1. **Docker** and **Docker Compose** installed on both the Grafana/Loki server and the server running Docker containers.
2. Access to the Docker logs directory (`/var/lib/docker/containers`) on the server running Docker containers.
3. Basic familiarity with Docker, Docker Compose, and Grafana.

---

## Architecture

- **Grafana/Loki Server**: Runs Grafana, Loki, and Promtail in Docker containers.
- **Remote Server**: Runs Docker containers and uses Promtail to ship logs to the Grafana/Loki server.

---

## Step 1: Set Up Grafana and Loki on the Main Server

### 1. Create a `docker-compose.yml` File

On the Grafana/Loki server, create a `docker-compose.yml` file with the following content:

```yaml
version: '3.7'

services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml

  promtail:
    image: grafana/promtail:latest
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/log:/var/log:ro
    command: -config.file=/etc/promtail/config.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - loki

volumes:
  loki-data:
  grafana-storage:

```
---

### 2. Start the Docker Compose Stack

Run the following command to start the stack:
```
docker-compose up -d
```

## 3. Access Grafana
Open your browser and go to `http://<grafana-server-ip>:3000`. Log in with:

```
Username: admin
Password: admin
```

## 4. Add Loki as a Data Source
Go to Configuration > Data Sources.
- Click Add data source.
- Select Loki.
- Set the URL to http://loki:3100.
- Click Save & Test.


## Step 2: Set Up Promtail on the Remote Server
1. Download Promtail
On the remote server, download and install Promtail:
```
curl -O -L "https://github.com/grafana/loki/releases/download/v2.8.0/promtail-linux-amd64.zip"
unzip promtail-linux-amd64.zip
chmod a+x promtail-linux-amd64
```
2. Create a `promtail-config.yml` File
Create a `promtail-config.yml` file with the following content:
```
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://<grafana-server-ip>:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*.log
    relabel_configs:
      - source_labels: ['__path__']
        regex: '/var/lib/docker/containers/([^/]+)/.+\.log'
        target_label: 'container_id'
```
Replace `<grafana-server-ip>` with the IP address of your Grafana/Loki server.

3. Run Promtail
Start Promtail with the following command:

```
./promtail-linux-amd64 -config.file=promtail-config.yml
```
## Step 3: Visualize Logs in Grafana
1. Go to Grafana
Open Grafana at `http://<grafana-server-ip>:3000`.

2. Query Logs
   1. Go to Explore.
   2. Select the Loki data source.
   3. Use the `{job="docker"}` query to view logs.

3. Filter by Container
Use the container_id label to filter logs by container:

```
{job="docker", container_id="<container-id>"}
```
