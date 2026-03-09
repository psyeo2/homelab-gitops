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
  - Includes Longhorn (`infra/storage/longhorn.yaml`) and placeholders for `cert-manager`, `ingress-nginx`, `postgres`, and `redis`.
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
