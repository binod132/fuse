Technical Document: Monitoring, Logging, and Alerting Implementation
Scenario Overview

To achieve end-to-end observability of the platform and ensure proactive issue detection, we deployed a comprehensive monitoring, logging, and alerting stack using Prometheus, Grafana, Loki, and Tempo.

1. Monitoring Setup
Prometheus

We installed the Prometheus stack via Helm with custom scrape configurations to monitor MySQL and NFS exporters alongside Kubernetes components.

Helm Commands
# Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Prometheus Stack with custom values
helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace -f values.yaml

# Install Loki for centralized logging
helm install loki grafana/loki-stack --namespace monitoring

# Install Tempo for distributed tracing
helm install tempo grafana/tempo --namespace monitoring

Prometheus Custom Scrape Configuration (values.yaml snippet)
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'mysql-exporter'
        static_configs:
          - targets: ['172.21.0.8:9104']
      - job_name: 'node-exporter'
        static_configs:
          - targets: ['172.21.0.9:9100']

grafana:
  enabled: true
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          access: proxy
          url: http://prometheus-operated.monitoring.svc.cluster.local:9090
          isDefault: false
        - name: Loki
          type: loki
          access: proxy
          url: http://loki.monitoring.svc.cluster.local:3100
          isDefault: false
        - name: Tempo
          type: tempo
          access: proxy
          url: http://tempo.monitoring.svc.cluster.local:3200
          isDefault: false

  adminPassword: "admin123"  # Optional: set Grafana admin password

persistence:
  enabled: true
  size: 10Gi
  storageClassName: "local-path"

2. Infrastructure Components (Docker Compose Sample)

For local testing and development, services were deployed using Docker Compose with attached networks and volumes:

networks:
  k3d-net:
    external: true
    name: k3d-dev-cluster  # replace with actual k3d network name

volumes:
  mysql-data:
  nfs-data:

services:
  mysql:
    image: mysql:8
    container_name: mysql2
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: appdb
      MYSQL_ROOT_HOST: "%"
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      - ./init_db.sql:/docker-entrypoint-initdb.d/init_db.sql
    networks:
      - k3d-net

  mysqld-exporter:
    image: prom/mysqld-exporter:v0.15.1
    container_name: mysqld-exporter
    restart: always
    environment:
      DATA_SOURCE_NAME: exporter_user:exporter_password@(mysql:3306)/
    ports:
      - "9104:9104"
    depends_on:
      - mysql
    networks:
      - k3d-net
    volumes:
      - ./.my.cnf:/.my.cnf:ro

  nfs-server:
    image: itsthenetwork/nfs-server-alpine:latest
    container_name: nfs-server
    restart: always
    environment:
      SHARED_DIRECTORY: /exports
    volumes:
      - nfs-data:/exports
    ports:
      - "20490:2049"     # NFS port
      - "11110:111"      # rpcbind port (required)
    networks:
      - k3d-net

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: always
    ports:
      - "9100:9100"
    networks:
      - k3d-net
    volumes:
      - mysql-data:/host/var/lib/mysql:ro
      - nfs-data:/host/exports:ro
    command:
      - '--path.rootfs=/host'

3. Dashboards & Alerts

Dashboards:
The Prometheus Helm chart deployed dashboards for Kubernetes cluster and pod monitoring out-of-the-box.
We built custom logging dashboards in Grafana using Loki, filtering logs by namespace and container labels for precise troubleshooting.
Grafana dashboards for MySQL and NFS metrics were created to monitor uptime and resource utilization.

Alerts:
Alerts for MySQL availability and NFS storage utilization (below 30% free space) were implemented with Prometheus alert rules and configured in Grafana. Alerts are firing properly based on the configured rules, but SMTP email notification setup is pending.

4. Alert Example: NFS Storage Utilization
groups:
- name: InfrastructureAlerts
  rules:
  - alert: NFSStorageLow
    expr: (node_filesystem_avail_bytes{mountpoint="/exports"} / node_filesystem_size_bytes{mountpoint="/exports"}) < 0.30
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Low NFS Storage on /exports"
      description: "Available storage on /exports dropped below 30%. Current free space: {{ $value | humanize1024 }}."

5. Tracing

Tempo was installed and integrated with Grafana as a datasource.

Application instrumentation to send traces using OpenTelemetry or similar is pending, enabling future distributed tracing and performance analysis.

Summary

Installed and configured Prometheus, Loki, and Tempo via Helm with extended scrape configs for MySQL and NFS exporters.

Deployed Grafana dashboards for Kubernetes, logs (via Loki), and infrastructure metrics.

Alerts configured and firing for critical components; email notifications setup is in progress.

Tempo installed, awaiting application-level tracing instrumentation.