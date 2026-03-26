# container-registry

Harbor is deployed here as an Argo CD `Application` using the official `goharbor/harbor` Helm chart.

Current bootstrap defaults:

- LAN exposure via MetalLB `LoadBalancer` on `192.168.1.57`
- `externalURL` set to `http://192.168.1.57`
- Longhorn-backed persistence for registry and job logs
- External PostgreSQL via `prodesks-postgres-rw.apps.svc.cluster.local`
- External Redis via `redis-master.apps.svc.cluster.local:6379`
- Trivy disabled for the initial bring-up
- Metrics and `ServiceMonitor` enabled for the existing Prometheus stack

## Before real use

Change these defaults before you rely on Harbor as a real registry:

- Replace `HARBOR_ADMIN_PASSWORD` in [sealedsecret.yaml](c:/Users/Eoin/Documents/Projects/laboratory/homelab-gitops/clusters/prodesks/apps/services/container-registry/sealedsecret.yaml).
- Replace `secretKey` in [sealedsecret.yaml](c:/Users/Eoin/Documents/Projects/laboratory/homelab-gitops/clusters/prodesks/apps/services/container-registry/sealedsecret.yaml) with your own stable 16-character value.
- Add `password` to [sealedsecret.yaml](c:/Users/Eoin/Documents/Projects/laboratory/homelab-gitops/clusters/prodesks/apps/services/container-registry/sealedsecret.yaml) for the `harbor_user` PostgreSQL password.
- Set a real hostname and TLS. Docker and Helm usage are much less painful once Harbor has a proper HTTPS URL.
