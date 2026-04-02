# traefik

Traefik is managed here as an Argo CD `Application` using the official Helm chart.

Current intent:

- dedicated namespace: `traefik`
- shared LAN VIP: `192.168.1.50`
- MetalLB-backed `LoadBalancer` service owned by the chart
- explicit ingress class: `prodesks-traefik`
- dashboard routes managed separately under `clusters/prodesks/infra/traefik/routes`

This keeps Traefik ownership out of the MetalLB config directory and avoids relying on an implicit k3s-bundled installation.

The route bundle is its own Argo CD `Application` so Traefik CRDs can be installed before `IngressRoute` resources are applied.
