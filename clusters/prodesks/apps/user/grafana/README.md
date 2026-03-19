# grafana

- App name: `prodesks-grafana`
- URL: `http://192.168.1.50:3000`
- Namespace: `apps`
- Helm chart: `grafana/grafana`
- Datasource: `Prometheus` at `monitoring-prometheus.monitoring.svc.cluster.local:9090`
- Default home dashboard: `Homelab Home` (`homelab-home-v2`)
- Dashboards:
  - `Node Exporter Full` (`gnetId: 1860`)
  - `NVIDIA DCGM Exporter` (`gnetId: 12239`)
  - `Cluster` (`gnetId: 15757`)
  - `Longhorn` (`gnetId: 16888`)
  - `Homelab Home (Legacy)` (provisioned from inline JSON)
  - `Homelab Home` (provisioned from inline JSON)

Admin credentials are stored in the chart-generated secret. After first sync:

```bash
kubectl -n apps get secret grafana -o jsonpath="{.data.admin-user}" | base64 -d
kubectl -n apps get secret grafana -o jsonpath="{.data.admin-password}" | base64 -d
```
