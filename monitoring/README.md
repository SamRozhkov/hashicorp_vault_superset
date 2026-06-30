# Kubernetes Cluster Monitoring Stack

This directory contains production-ready monitoring configuration for the K8s cluster.

## Components

### 1. Prometheus Stack (`prometheus-argocd-app.yaml`)
- **Prometheus**: Time-series metrics database
- **Grafana**: Visualization and dashboards
- **Alertmanager**: Alert routing and notification
- **kube-state-metrics**: Kubernetes resource state metrics
- **node-exporter**: Node-level metrics (CPU, memory, disk)

**Configuration:**
- 15-day retention for Prometheus
- 50GB storage for metrics
- Vault metrics scraping enabled
- Slack integration for alerts

**Before applying:**
1. Replace `YOUR_VAULT_TOKEN` with actual Vault token
2. Replace `YOUR_SLACK_WEBHOOK_URL` with your Slack webhook
3. Adjust storage sizes based on cluster needs

### 2. Loki Stack (`loki-argocd-app.yaml`)
- **Loki**: Log aggregation (Prometheus-like for logs)
- **Promtail**: Log collector (agent)
- Automatically scrapes logs from all K8s pods

**Configuration:**
- 30GB storage for logs
- 168-hour log retention

### 3. Alert Rules (`prometheus-rules.yaml`)
Critical alerts for:
- Node health (NotReady, MemoryPressure, DiskPressure)
- Pod crashes and unhealthy states
- Deployment replica mismatches
- Volume capacity warnings
- Vault sealed state
- High auth failures
- API server latency

### 4. Service Monitors (`service-monitor-vault.yaml`)
Auto-discovery of metrics from:
- Vault (metrics endpoint)
- Airflow (if metrics enabled)
- Superset (if metrics exposed)

### 5. Grafana Dashboards (`grafana-dashboards.yaml`)
Pre-built dashboards for:
- Kubernetes cluster overview
- Vault monitoring

## Deployment Instructions

1. **Create monitoring namespace:**
   ```bash
   kubectl apply -f monitoring/namespace.yaml
   ```

2. **Apply via ArgoCD:**
   ```bash
   kubectl apply -f monitoring/prometheus-argocd-app.yaml
   kubectl apply -f monitoring/loki-argocd-app.yaml
   ```

3. **Create alert rules and service monitors:**
   ```bash
   kubectl apply -f monitoring/prometheus-rules.yaml
   kubectl apply -f monitoring/service-monitor-vault.yaml
   ```

4. **Access Grafana:**
   - Port-forward: `kubectl port-forward -n monitoring svc/prometheus-stack-grafana 3000:80`
   - Default credentials: admin / admin
   - Or expose via ingress

5. **Configure Alerts:**
   - Update Slack webhook in `prometheus-argocd-app.yaml`
   - Configure email/PagerDuty/Teams as needed

## Post-Install Configuration

### 1. Update Alertmanager
Edit the AlertManager config to add channels:
```bash
kubectl edit configmap alertmanager-config -n monitoring
```

### 2. Add Custom Alert Rules
Create additional PrometheusRule resources for application-specific alerts.

### 3. Configure Grafana Data Sources
Grafana automatically discovers Prometheus. Add Loki data source:
- URL: `http://loki:3100`
- Type: Loki

### 4. Vault Token for Metrics
Create a Vault policy and token for metrics scraping:
```bash
vault policy write prometheus - <<EOF
path "sys/metrics" {
  capabilities = ["read"]
}
EOF

vault token create -policy=prometheus -ttl=8760h
```

## Monitoring Best Practices

1. **Retention Policy**: Adjust based on compliance/storage needs
2. **Alert Fatigue**: Start with fewer alerts, refine over time
3. **Dashboard Organization**: Separate by team/service
4. **Regular Reviews**: Monthly review of alert effectiveness
5. **Backup**: Consider backing up Grafana dashboards

## Troubleshooting

**Prometheus not scraping metrics:**
```bash
kubectl port-forward -n monitoring svc/prometheus-stack-kube-prom-prometheus 9090:9090
# Visit http://localhost:9090/targets
```

**Grafana empty dashboards:**
- Verify Prometheus data source is connected
- Check metric names with `kubectl port-forward` and Prometheus UI

**Missing pod logs:**
- Verify Promtail pods are running
- Check Promtail config: `kubectl logs -n monitoring -l app=promtail`

## Costs

- Prometheus: ~5GB/week per 100 pods
- Loki: ~1GB/week per 100 pods
- Adjust retention/sampling to fit your infrastructure
