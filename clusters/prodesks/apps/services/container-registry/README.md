# container-registry

Harbor is deployed here as an Argo CD `Application` using the official `goharbor/harbor` Helm chart.

Current bootstrap defaults:

- LAN exposure via MetalLB `LoadBalancer` on `192.168.1.57`
- `externalURL` set to `http://192.168.1.57`
- Longhorn-backed persistence for registry, job logs, internal PostgreSQL, and internal Redis
- Trivy disabled for the initial bring-up
- Metrics and `ServiceMonitor` enabled for the existing Prometheus stack

## Why internal DB and Redis for now

This bootstrap keeps Harbor self-contained so it can come up immediately.

Two details in the current shared-services setup make external wiring awkward on the first pass:

- Harbor expects an external Redis password secret key named `REDIS_PASSWORD`, but the existing Redis secret is keyed as `REDIS_DEFAULT_PASS`.
- The current CloudNativePG cluster only bootstraps the generic `app` database, and this repo does not yet declaratively provision a dedicated Harbor database/user.

## Before real use

Change these defaults before you rely on Harbor as a real registry:

- Replace `HARBOR_ADMIN_PASSWORD` in [sealedsecret.yaml](c:/Users/Eoin/Documents/Projects/laboratory/homelab-gitops/clusters/prodesks/apps/services/container-registry/sealedsecret.yaml).
- Replace `secretKey` in [sealedsecret.yaml](c:/Users/Eoin/Documents/Projects/laboratory/homelab-gitops/clusters/prodesks/apps/services/container-registry/sealedsecret.yaml) with your own stable 16-character value.
- Set a real hostname and TLS. Docker and Helm usage are much less painful once Harbor has a proper HTTPS URL.

## Likely next step

Once Harbor is up, the next cleanup pass is:

- give Harbor a proper external hostname and TLS
- optionally migrate Harbor to shared Postgres and Redis once the secret keys and database provisioning are aligned
