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
  - Includes Longhorn (`infra/storage/longhorn.yaml`) and placeholders for `cert-manager`, `traefik`, `postgres`, and `redis`.
  - Includes shared networking resources (`infra/networking`).
- `clusters/prodesks/apps`:
  - User-facing workloads.

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

## Ingress (Traefik)

- For k3s, Traefik is a sensible default ingress controller.
- `ingress-nginx` is still actively maintained, but running both usually adds unnecessary complexity for homelab setups.
- This repo includes UI ingresses for:
  - Longhorn: `longhorn.prodesks.lan`
  - Argo CD: `argocd.prodesks.lan`
- Manifests:
  - `clusters/prodesks/infra/networking/longhorn-ui-ingress.yaml`
  - `clusters/prodesks/infra/networking/argocd-ui-ingress.yaml`
- To access from your LAN, create local DNS/hosts entries for both hosts to any k3s node IP that serves Traefik on port `80`.

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
