# immich

Immich is deployed as an Argo CD `Application` (`immich.yaml`) using the official Helm chart.

## What this setup does

- Uses your existing CloudNativePG cluster at `prodesks-postgres-rw.apps.svc.cluster.local`.
- Uses chart-managed Valkey (`valkey.enabled: true`).
- Mounts the photo library from an NFS-backed PVC (`immich-library`) sourced from your NAS (`192.168.1.165`).
- Mounts a second read-only external media PVC at `/external` for existing NAS content.
- Exposes Immich server on MetalLB IP `192.168.1.54` port `2283`.
- Keeps in-cluster ML disabled for now (`machine-learning.enabled: false`) so you can wire external ML later.

## External libraries

- Uploads are stored on the `immich-library` PVC (`/var/nfs/shared/Photos/immich`).
- Existing NAS media is mounted read-only at `/external` from `/var/nfs/shared/Photos`.
- In Immich UI, add external libraries as `/external/photos` and `/external/videos`.

## Library layout on NAS

- The NFS PV points to `/var/nfs/shared/Photos`.
- Immich uploads are stored under `/data/immich/uploads` inside the container (NAS path `/var/nfs/shared/Photos/immich/uploads`).
- External libraries should be added in Immich as `/external/photos` and `/external/videos`.
- Do not add `/data` or `/data/immich` as external libraries.

## Important values to verify

- `immich-library-pv.yaml`:
  - `nfs.server` should be `192.168.1.165`
  - `nfs.path` must match your real NAS export path
- `immich.yaml`:
  - `image.tag` (currently `v1.133.0`)
  - `server.service.main.loadBalancerIP` (currently `192.168.1.54`)

## After Deployment

- Add external libraries as above
- Set Immich Machine Learning URL to `http://192.168.1.161:3003`