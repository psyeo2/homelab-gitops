# grafana

Deploys the Prometheus community `kube-prometheus-stack` chart into the `apps`
namespace and exposes Grafana on the shared LAN VIP:

- Grafana: `http://192.168.1.50:3000`

Included in this app:

- Prometheus Operator CRDs via a dedicated Argo CD app
- Prometheus with Longhorn-backed storage
- Grafana with Longhorn-backed storage
- Default kube-prometheus dashboards
- Custom `Homelab Nodes + GPU` dashboard
- External scrape job for GPU metrics from `192.168.1.161:9400`

Notes:

- This is tuned for the `prodesks` k3s cluster, so controller-manager,
  scheduler, etcd, and kube-proxy scraping are disabled.
- The CRDs are split into `prometheus-operator-crds` first, then the main
  `kube-prometheus-stack` app syncs after them. This avoids Argo CD ordering
  issues around Prometheus Operator custom resources.
- GPU panels assume a DCGM-style exporter exposing metrics such as
  `DCGM_FI_DEV_GPU_UTIL`.
