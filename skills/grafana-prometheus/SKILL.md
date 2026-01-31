# Monitoring Stack: Grafana + Prometheus

Complete observability for your homelab.

## When to Use

- Monitor server resources (CPU, RAM, disk)
- Track container metrics
- Visualize network traffic
- Set up alerting for issues

## Architecture

```
Exporters → Prometheus (scrape & store) → Grafana (visualize)
                    ↓
              Alertmanager (notify)
```

## Docker Compose Stack

```yaml
networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=90d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=changeme
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3000:3000"
    networks:
      - monitoring
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring
    restart: unless-stopped
```

## Prometheus Configuration

### prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
        labels:
          instance: 'homelab-server'

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'traefik'
    static_configs:
      - targets: ['traefik:8082']

  # Additional hosts
  - job_name: 'remote-nodes'
    static_configs:
      - targets: ['192.168.1.20:9100', '192.168.1.30:9100']
        labels:
          group: 'servers'
```

## Exporters

### Node Exporter (System Metrics)

Deploy on every machine:
```yaml
node-exporter:
  image: prom/node-exporter:latest
  ports:
    - "9100:9100"
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    - /:/rootfs:ro
  command:
    - '--path.procfs=/host/proc'
    - '--path.sysfs=/host/sys'
    - '--path.rootfs=/rootfs'
```

### cAdvisor (Container Metrics)

Already in main stack - monitors all Docker containers.

### Blackbox Exporter (Endpoint Probing)

```yaml
blackbox-exporter:
  image: prom/blackbox-exporter:latest
  ports:
    - "9115:9115"
  volumes:
    - ./blackbox.yml:/etc/blackbox_exporter/config.yml
```

`blackbox.yml`:
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: []
      method: GET
```

`prometheus.yml`:
```yaml
- job_name: 'blackbox'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://example.com
        - https://jellyfin.example.com
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

## Grafana Dashboards

### Provisioning

`grafana/provisioning/datasources/prometheus.yml`:
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

### Recommended Dashboards

Import by ID in Grafana:
| ID | Name | Use |
|----|------|-----|
| 1860 | Node Exporter Full | Server metrics |
| 193 | Docker Dashboard | Container stats |
| 13659 | Blackbox Exporter | Endpoint uptime |
| 11462 | Traefik | Reverse proxy |
| 7249 | Cadvisor | Container resources |

### Import Dashboard

1. Grafana → Dashboards → Import
2. Enter Dashboard ID
3. Select Prometheus data source
4. Import

## Alert Rules

### rules/alerts.yml

```yaml
groups:
  - name: homelab
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | printf \"%.1f\" }}%"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory on {{ $labels.instance }}"

      - alert: DiskSpaceLow
        expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100 > 85
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"

      - alert: ContainerDown
        expr: absent(container_last_seen{name=~".*"}) or time() - container_last_seen > 60
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container {{ $labels.name }} is down"

      - alert: ServiceDown
        expr: probe_success == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} is unreachable"
```

## Alertmanager

### docker-compose addition

```yaml
alertmanager:
  image: prom/alertmanager:latest
  container_name: alertmanager
  volumes:
    - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
  ports:
    - "9093:9093"
  networks:
    - monitoring
  restart: unless-stopped
```

### alertmanager.yml

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'discord'

receivers:
  - name: 'discord'
    discord_configs:
      - webhook_url: 'https://discord.com/api/webhooks/...'
        title: '{{ .GroupLabels.alertname }}'
        message: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: 'email'
    email_configs:
      - to: 'admin@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'password'
```

## Useful PromQL Queries

### CPU Usage
```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Memory Usage
```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

### Disk Usage
```promql
(1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes) * 100
```

### Container CPU
```promql
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```

### Network Traffic
```promql
rate(node_network_receive_bytes_total{device="eth0"}[5m])
rate(node_network_transmit_bytes_total{device="eth0"}[5m])
```

## Best Practices

1. **Set retention** - 90 days is usually enough
2. **Use labels** - group and filter metrics
3. **Start with pre-built dashboards** - customize later
4. **Alert on symptoms** - not just thresholds
5. **Test alerts** - fire test alerts regularly
6. **Backup Grafana** - dashboards and provisioning
7. **Remote write** - for long-term storage (optional)
