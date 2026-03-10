# homelab-gitops

GitOps source of truth for the `prodesks` k3s cluster, managed by Argo CD.

## Bootstrap flow

Argo CD needs one manually-created app to start reconciliation.  
That app is the root app in this repo:

- `clusters/prodesks/bootstrap/root-application.yaml`

The root app then manages child apps from:

- `clusters/prodesks/argocd/applications`

Current child apps:

- `prodesks-infra` -> `clusters/prodesks/infra`
- `prodesks-apps` -> `clusters/prodesks/apps`

## Directory layout

- `clusters/prodesks/bootstrap`:
  - Root app bootstrap manifest (manually applied once).
- `clusters/prodesks/argocd/applications`:
  - Argo CD `Application` CRs only.
- `clusters/prodesks/infra`:
  - Platform services and cluster-level dependencies.
  - Includes Longhorn (`infra/storage/longhorn.yaml`) plus platform networking/load-balancer resources.
  - Includes shared networking resources (`infra/networking`).
- `clusters/prodesks/apps`:
  - Application workloads split by role:
  - `apps/services`: shared internal dependencies (postgres, redis, ddns, etc.).
  - `apps/user`: user-facing apps and user utilities (immich, game servers, wireguard, etc.).

## One-time setup

1. Confirm these values in all Application manifests:
   - `spec.source.repoURL` is your repo URL in Argo CD.
   - `spec.destination.name` matches your registered cluster name in Argo CD (currently `in-cluster`).
2. Create the root app once:
   - With `kubectl`:
     - `kubectl apply -f clusters/prodesks/bootstrap/root-application.yaml`
   - Or via Argo CD UI:
     - Create app `prodesks-root` with path `clusters/prodesks/argocd/applications`.
3. Sync `prodesks-root` in Argo CD.

After that, new apps/resources only need Git commits.

## Longhorn first

- Longhorn is deployed by `prodesks-infra` from `clusters/prodesks/infra/storage/longhorn.yaml`.
- It is configured to:
  - Use `/var/lib/longhorn` as the data path.
  - Use k3s kubelet path `/var/lib/rancher/k3s/agent/kubelet`.
  - Create Longhorn as the default storage class with replica count `3`.
- Sync ordering is set so `prodesks-infra` runs before `prodesks-apps`.

Before syncing, make sure each node has Longhorn prerequisites installed (notably `open-iscsi`/`iscsiadm`) and running.

## UI Access (LAN VIP)

- This repo exposes UI services directly via a shared MetalLB VIP:
  - Argo CD: `https://192.168.1.50:8080`
  - Longhorn: `http://192.168.1.50:8081`
- Manifests:
  - `clusters/prodesks/infra/networking/argocd-ui-lb.yaml`
  - `clusters/prodesks/infra/networking/longhorn-ui-lb.yaml`
- Both services share the same VIP using `metallb.io/allow-shared-ip`.

## LoadBalancer IP (MetalLB)

- MetalLB is managed by:
  - `clusters/prodesks/infra/metallb/metallb.yaml`
  - `clusters/prodesks/infra/metallb/metallb-config.yaml`
- Config path:
  - `clusters/prodesks/infra/metallb/metallb-config`
- Current LAN pool:
  - `192.168.1.50-192.168.1.59`
- Traefik is pinned to VIP:
  - `192.168.1.50` via `metallb.io/loadBalancerIPs` annotation.

Important for k3s:
- Disable built-in `servicelb` when using MetalLB as the load balancer controller to avoid conflicts.

## Sealed Secrets

- Controller is managed by:
  - `clusters/prodesks/infra/sealed-secrets/sealed-secrets.yaml`
- Namespace:
  - `sealed-secrets`
- Use SealedSecrets for API keys and app credentials so plaintext secrets are never committed to Git.

## Cloudflare DDNS

- Managed in:
  - `clusters/prodesks/apps/services/cloudflare-ddns`
- Purpose:
  - Keep `ddns.diakonos.uk` (or your chosen host) updated to your current WAN/public IP.
- Notes:
  - This is separate from MetalLB VIPs (`192.168.1.x`) used inside your LAN.
