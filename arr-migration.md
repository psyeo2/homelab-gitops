# ARR Migration Plan

## Goal

Rebuild the ARR stack so that:

- all ARR apps run in the `arr` namespace instead of `apps`
- web access goes through Traefik and Pi-hole hostnames instead of MetalLB VIPs
- all web-facing ARR services become `ClusterIP`
- qBittorrent remains isolated from the rest of the stack
- all apps except qBittorrent are grouped behind a shared `gluetun` sidecar
- existing config is preserved by migration where possible rather than delete/recreate

## Current State

Current manifests live under `clusters/prodesks/apps/user/arr`.

Current split:

- `prodesks-arr-qbittorrent`
  - `qBittorrent + Prowlarr + gluetun`
  - namespace `apps`
  - exposed as `LoadBalancer` on `192.168.1.55`
  - ports:
    - qBittorrent Web UI `8080`
    - qBittorrent torrent port `6881/TCP+UDP`
    - Prowlarr `9696`
- `prodesks-arr-services`
  - `Sonarr`, `Radarr`, `Lidarr`, `Bazarr`, `Jellyseerr`
  - namespace `apps`
  - exposed as `LoadBalancer` shared on `192.168.1.56`

Current storage:

- `qbittorrent-config`, `prowlarr-config`, `sonarr-config`, `radarr-config`, `lidarr-config`, `bazarr-config`, `jellyseerr-config`
  - namespaced PVCs in `apps`
  - mostly `ReadWriteOnce`
  - cannot be moved in place to a different namespace
- `jellyfin-media`
  - namespaced PVC in `apps`
  - backed by static NFS PV `jellyfin-media-nfs-pv`
  - `ReadWriteMany`
  - can be recreated in `arr` against the same PV

Current routing:

- Traefik ingresses point at ARR services in namespace `apps`
- Homepage already links to `.home.arpa` routes

## Target State

Target split:

- `prodesks-arr-qbittorrent`
  - namespace `arr`
  - `qBittorrent + gluetun`
  - Web UI exposed via Traefik
  - torrent port `6881/TCP+UDP` kept separately exposed during first migration for safety
- `prodesks-arr-services`
  - namespace `arr`
  - `Prowlarr + Sonarr + Radarr + Lidarr + Bazarr + Jellyseerr + gluetun`
  - all web access via Traefik only
  - no MetalLB VIPs

Target service types:

- all web services: `ClusterIP`
- qBittorrent torrent port:
  - keep separately exposed for first cutover using a dedicated service
  - do not rely on Traefik for torrent peer traffic

Target ingress routing:

- `qbittorrent.home.arpa`
- `prowlarr.home.arpa`
- `sonarr.home.arpa`
- `radarr.home.arpa`
- `lidarr.home.arpa`
- `bazarr.home.arpa`
- `jellyseerr.home.arpa`

All of the above should point to services in namespace `arr`.

## Assumptions To Preserve

Mount paths should remain unchanged unless explicitly revisited:

- qBittorrent, Prowlarr, Sonarr, Radarr, Lidarr, Bazarr:
  - config at `/config`
- Jellyseerr:
  - config at `/app/config`
- shared media/downloads:
  - mounted at `/data`

These paths must be checked during implementation before final cutover.

## Important Constraints

### PVC migration constraint

The config PVCs are namespace-scoped and mostly `ReadWriteOnce`. Because of that:

- they cannot simply be renamed or moved from `apps` to `arr`
- preserving config requires:
  - creating replacement PVCs in `arr`
  - migrating data from old PVCs to new PVCs using a method that works across namespaces
  - performing cutover during downtime

Additional implementation note discovered during execution:

- a Pod cannot mount PVCs from two different namespaces
- this means the naive "single copy job mounts old `apps` PVC and new `arr` PVC" approach is invalid
- data migration will need one of:
  - Longhorn snapshot/clone workflow
  - backup/restore workflow
  - a host-level/manual copy process outside a single in-cluster pod

### qBittorrent networking constraint

Traefik is only for HTTP/HTTPS app access.

qBittorrent has two different traffic types:

- Web UI over HTTP, which can be moved behind Traefik
- BitTorrent peer traffic on `6881/TCP+UDP`, which is not standard Traefik ingress traffic

For first migration, use the safer approach:

- move qBittorrent Web UI behind Traefik
- keep `6881/TCP+UDP` externally exposed using a dedicated service
- revisit removal later after verifying no regression

### SealedSecret migration constraint

The current ARR VPN key is stored as a `SealedSecret` scoped to namespace `apps`.

Additional implementation note discovered during execution:

- the existing encrypted payload cannot be moved to namespace `arr` by changing YAML only
- it must be resealed for namespace `arr`, or recreated by another secure secret delivery path
- do not switch ARR applications to namespace `arr` until this secret migration is solved

## Planned Repository Changes

### Namespaces

- add new namespace manifest for `arr`
- include it in infra namespace kustomization if not already present

### ARR manifests

Refactor `clusters/prodesks/apps/user/arr` so that:

- all resources target namespace `arr`
- `sealedsecret.yaml` is recreated for namespace `arr`
- `jellyfin-media` PVC is recreated in `arr`
- all config PVC manifests are recreated in `arr`

### Applications

Reshape applications into:

- `prodesks-arr-qbittorrent`
  - `qBittorrent + gluetun`
  - `ClusterIP` for Web UI
  - separate service for `6881/TCP+UDP`
- `prodesks-arr-services`
  - `Prowlarr + Sonarr + Radarr + Lidarr + Bazarr + Jellyseerr + gluetun`
  - all `ClusterIP`

### Traefik

Update `clusters/prodesks/infra/traefik/routes/apps-dashboard-routes.yaml` so that:

- ARR ingresses move from namespace `apps` to namespace `arr`
- backend service names are updated if service names change during refactor

### Homepage

Homepage already points at `.home.arpa` routes for ARR UIs, so no ARR URL migration should be required unless hostnames or paths change.

## Planned Execution Sequence

### Phase 1: Prepare namespace and storage

1. Create namespace `arr`.
2. Recreate ARR secret in `arr`.
3. Create `jellyfin-media` PVC in `arr` using the same NFS-backed PV.
4. Create new config PVCs in `arr`:
   - `qbittorrent-config`
   - `prowlarr-config`
   - `sonarr-config`
   - `radarr-config`
   - `lidarr-config`
   - `bazarr-config`
   - `jellyseerr-config`

### Phase 2: Define new workloads

1. Update qBittorrent application:
   - namespace `arr`
   - remove Prowlarr from this release
   - keep `gluetun`
   - expose Web UI internally only
   - expose torrent port separately for safety
2. Update ARR services application:
   - namespace `arr`
   - add `Prowlarr` to this release
   - add `gluetun`
   - convert all web services to `ClusterIP`

### Phase 3: Data migration

Run one-off copy jobs while old apps are stopped.

For each config PVC:

1. stop old workload that owns the PVC
2. mount old `apps` PVC and new `arr` PVC into a migration pod
3. copy data
4. verify copied contents

PVC copy pairs:

- `apps/qbittorrent-config` -> `arr/qbittorrent-config`
- `apps/prowlarr-config` -> `arr/prowlarr-config`
- `apps/sonarr-config` -> `arr/sonarr-config`
- `apps/radarr-config` -> `arr/radarr-config`
- `apps/lidarr-config` -> `arr/lidarr-config`
- `apps/bazarr-config` -> `arr/bazarr-config`
- `apps/jellyseerr-config` -> `arr/jellyseerr-config`

### Phase 4: Routing cutover

1. Update Traefik ingress manifests to point to namespace `arr`.
2. Sync Argo applications.
3. Verify:
   - all ARR UIs load at `.home.arpa`
   - qBittorrent Web UI works through Traefik
   - qBittorrent torrent port service still exists
   - app configs and libraries are intact

### Phase 5: Cleanup

After verification:

1. remove old ARR resources in namespace `apps`
2. remove old `apps` ARR PVCs only after confirming rollback is no longer needed
3. optionally revisit qBittorrent torrent port exposure later

## Verification Checklist

After migration, verify:

- `kubectl get pods -n arr`
- `kubectl get svc -n arr`
- `kubectl get ingress -A | grep -E 'qbittorrent|prowlarr|sonarr|radarr|lidarr|bazarr|jellyseerr'`
- `curl -k -I --resolve qbittorrent.home.arpa:443:192.168.1.50 https://qbittorrent.home.arpa/`
- `curl -k -I --resolve prowlarr.home.arpa:443:192.168.1.50 https://prowlarr.home.arpa/`
- `curl -k -I --resolve sonarr.home.arpa:443:192.168.1.50 https://sonarr.home.arpa/`
- `curl -k -I --resolve radarr.home.arpa:443:192.168.1.50 https://radarr.home.arpa/`
- `curl -k -I --resolve lidarr.home.arpa:443:192.168.1.50 https://lidarr.home.arpa/`
- `curl -k -I --resolve bazarr.home.arpa:443:192.168.1.50 https://bazarr.home.arpa/`
- `curl -k -I --resolve jellyseerr.home.arpa:443:192.168.1.50 https://jellyseerr.home.arpa/`

Functional verification:

- qBittorrent:
  - existing torrents present
  - Web UI accessible
  - torrent port still exposed
- Prowlarr:
  - indexers preserved
- Sonarr/Radarr/Lidarr/Bazarr:
  - libraries preserved
  - root folders still valid
  - download client settings still valid
- Jellyseerr:
  - app config preserved
  - auth and request flow still works

## Rollback Strategy

Rollback should be possible until old `apps` PVCs and applications are removed.

Rollback path:

1. scale down or disable the new `arr` workloads
2. restore old ingress targets if needed
3. re-enable old `apps` ARR applications
4. keep old PVCs untouched until migration is verified complete

Do not delete old PVCs during initial cutover.

## Implementation Notes

- Do not execute this migration by changing namespace fields in place on existing live objects.
- Prefer additive migration:
  - create new namespace
  - create new PVCs
  - copy data
  - cut traffic over
  - remove old resources last
- Check each app's internal path configuration before final apply, especially:
  - download paths
  - root/library paths
  - qBittorrent save path/category paths
  - Sonarr/Radarr download client mappings

## Open Question For Execution

Before implementation, confirm whether the qBittorrent torrent port should stay as:

- `LoadBalancer` on first cutover

or

- another service shape that still preserves `6881/TCP+UDP`

The documented plan assumes the safer first option: keep the torrent port externally exposed during initial migration.

## Execution Status

Started in repo:

- added namespace manifest `clusters/prodesks/infra/namespaces/arr.yaml`
- added `arr` namespace to infra namespace kustomization

Blocked before application cutover:

- ARR VPN `SealedSecret` must be resealed for namespace `arr`
- ARR config PVC migration method must be chosen because cross-namespace copy jobs are not valid
