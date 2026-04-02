# traefik

This directory configures the bundled k3s Traefik deployment instead of installing a second controller.

Current intent:

- keep the native `kube-system/traefik`
- configure its service via `HelmChartConfig`
- keep the shared LAN VIP at `192.168.1.50`
- route app hostnames with plain Kubernetes `Ingress` resources

Files here:

- `traefik-config.yaml`: k3s `HelmChartConfig` for the packaged Traefik release
- `routes/`: hostname-to-service ingress rules

This avoids fighting the built-in k3s Traefik service and avoids depending on Traefik CRD API versions for routing.
