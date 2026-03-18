# grafana

- Grafana: `http://192.168.1.50:3000`
- Namespace: `monitoring`
- Helm chart: `prometheus-community/kube-prometheus-stack`
- Extra dashboard: `Node Exporter Full` from the upstream `rfmoz/grafana-dashboards` repository

Admin credentials are stored in the chart-generated secret. After first sync:

```bash
kubectl -n monitoring get secret monitoring-grafana -o jsonpath="{.data.admin-user}" | base64 -d
kubectl -n monitoring get secret monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```
