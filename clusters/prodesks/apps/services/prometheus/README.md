# prometheus

- App name: `prodesks-prometheus`
- Path: `clusters/prodesks/apps/services/prometheus`
- Namespace: `monitoring`
- Helm chart: `prometheus-community/kube-prometheus-stack`
- Purpose: scrape and store Kubernetes metrics for the cluster

This app intentionally disables Grafana so Prometheus health is visible separately in Argo CD.
