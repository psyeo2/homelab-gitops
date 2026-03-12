# immich

Immich is deployed as an Argo CD `Application` (`immich.yaml`) using the official Helm chart.

## What this setup does

- Uses your existing CloudNativePG cluster at `prodesks-postgres-rw.apps.svc.cluster.local`.
- Uses chart-managed Valkey (`valkey.enabled: true`).
- Mounts the photo library from an NFS-backed PVC (`immich-library`) sourced from your NAS (`192.168.1.165`).
- Exposes Immich server on MetalLB IP `192.168.1.54` port `2283`.
- Keeps in-cluster ML disabled for now (`machine-learning.enabled: false`) so you can wire external ML later.

## Important values to verify

- `immich-library-pv.yaml`:
  - `nfs.server` should be `192.168.1.165`
  - `nfs.path` must match your real NAS export path
- `immich.yaml`:
  - `image.tag` (currently `v1.133.0`)
  - `server.service.main.loadBalancerIP` (currently `192.168.1.54`)

## Deploy

```bash
argocd app sync prodesks-apps
kubectl -n apps get pvc immich-library
kubectl -n apps get pods | grep immich
kubectl -n apps get svc | grep immich
```
