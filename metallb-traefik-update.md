# MetalLB, Traefik, and LAN Network Structure

## Current Position

The cluster already uses MetalLB to expose `LoadBalancer` services on the LAN.

- MetalLB itself is installed by Argo CD from the official Helm chart.
- MetalLB configuration is applied separately as Kubernetes custom resources.
- Traefik is intended to sit on a shared LAN VIP at `192.168.1.50`.
- Some services are still exposed directly as `LoadBalancer` services on `IP:port`.

This works, but the structure is currently a mix of:

- MetalLB controller installation
- MetalLB pool/advertisement config
- consumer services that ask MetalLB for a LAN IP

That is why the repo currently feels more convoluted than the underlying idea really is.

## What MetalLB Does

MetalLB gives Kubernetes `LoadBalancer` services a real IP address on the LAN.

For this cluster, the important pieces are:

- `IPAddressPool`: defines the LAN address range MetalLB may allocate from
- `L2Advertisement`: tells MetalLB to advertise those IPs on the local network

Example pool already present:

- `192.168.1.50-192.168.1.59`

Important point:

MetalLB does not do hostname routing and it does not do DNS. It only makes Kubernetes services reachable at LAN IPs.

## What Traefik Does

Traefik is the HTTP/HTTPS ingress router.

Its job is:

- listen on one shared IP, ideally `192.168.1.50`
- receive web requests for many hostnames
- inspect the request hostname
- route each request to the correct internal Kubernetes service and port

Traefik does not provide DNS records. It routes traffic after the client has already resolved a hostname and connected to the Traefik IP.

## What Pi-hole Does

Pi-hole is the DNS layer for the LAN.

Its job in the new structure is:

- answer queries for local service names
- return the Traefik VIP `192.168.1.50`

Examples:

- `argocd.home.arpa` -> `192.168.1.50`
- `grafana.home.arpa` -> `192.168.1.50`
- `homeassistant.home.arpa` -> `192.168.1.50`

Pi-hole does not proxy the request to Traefik. The client resolves the name via Pi-hole, then connects directly to Traefik at the returned IP.

## End-to-End Request Flow

The intended flow is:

1. A client asks Pi-hole for `grafana.home.arpa`
2. Pi-hole returns `192.168.1.50`
3. The client connects to `https://192.168.1.50`
4. The HTTP request includes the hostname `grafana.home.arpa`
5. Traefik matches that hostname and forwards the request to the correct Kubernetes service

So the responsibilities are:

- Pi-hole: hostname -> Traefik IP
- Traefik: hostname -> Kubernetes service and port
- MetalLB: make the Traefik `LoadBalancer` IP exist on the LAN

## Intended New LAN Structure

The target model is:

- one main LAN VIP for Traefik: `192.168.1.50`
- web apps exposed through Traefik using hostnames instead of raw ports
- most web-facing workloads converted to `ClusterIP` behind Traefik
- Pi-hole local DNS records pointing service names at the Traefik VIP

This replaces access patterns like:

- `https://192.168.1.50:8080`
- `http://192.168.1.50:8081`
- `http://192.168.1.54:2283`

with cleaner access patterns like:

- `https://argocd.home.arpa`
- `https://longhorn.home.arpa`
- `https://immich.home.arpa`

## What Should Go Behind Traefik

Traefik is a good fit for:

- web UIs
- browser-based apps
- HTTP APIs
- HTTPS services that should share one LAN IP

Examples in this repo likely suited to that model:

- Argo CD
- Longhorn UI
- Grafana
- Homepage
- Home Assistant
- Vaultwarden
- Immich
- Kiwix
- ClankAssist HTTP services

## What Should Probably Not Go Behind Traefik by Default

Traefik solves ingress for HTTP and HTTPS. It does not automatically simplify arbitrary TCP or UDP services.

These often remain better as direct `LoadBalancer` services, node ports, VPN-only access, or internal-only services:

- Postgres
- Redis
- MQTT
- WireGuard
- BitTorrent-related ports
- other raw TCP/UDP protocols

Some of these can be routed through Traefik using TCP/UDP entrypoints, but that is a separate design and should not be mixed into the basic web-ingress cleanup.

## DNS Naming Guidance

Do not use `.local`.

Reason:

- `.local` is reserved in practice for mDNS / Bonjour and often causes confusion or inconsistent resolution on home networks

Better choices:

- `.home.arpa` for purely internal LAN names
- a real domain you already own, using split DNS if you want internal and external resolution to differ

Recommended internal examples:

- `argocd.home.arpa`
- `grafana.home.arpa`
- `ha.home.arpa`

## HTTPS Guidance

If services are exposed as `https://...`, certificates need to match the hostname and be trusted by client devices.

For purely internal names like `.home.arpa`, public certificate authorities will not issue certificates.

That means LAN HTTPS usually requires one of:

- an internal CA that your devices trust
- a tool such as `step-ca`
- a real domain you own, with a certificate strategy that fits your internal/external DNS model

Without this, HTTPS may work technically but clients will show certificate warnings.

## Ingress vs Egress

This redesign is mainly about ingress, not egress.

- ingress: traffic coming into the cluster from clients on the LAN
- egress: traffic leaving pods for the internet or other networks

Traefik helps with ingress.
It does not materially redesign outbound pod traffic.

## Why the Current MetalLB Layout Has So Many Files

The current file count comes from several layers being split apart:

1. MetalLB installation
2. MetalLB configuration
3. Kustomize assembly
4. individual MetalLB CRs such as `IPAddressPool` and `L2Advertisement`
5. consumer service manifests that request specific LAN IPs

That split is partly normal GitOps structure and partly upstream MetalLB design.

What is arguably awkward in the current repo is that the Traefik `LoadBalancer` service lives under MetalLB config, even though it is not really MetalLB configuration. It is a service that consumes MetalLB.

## Helm Chart Note

The repo already uses the official MetalLB Helm chart for installation.

However, upstream MetalLB still expects real configuration objects such as:

- `IPAddressPool`
- `L2Advertisement`

So Helm does not fully remove the need for separate manifests.

A custom wrapper chart could template those resources too, but that would mainly reduce repo sprawl, not reduce the number of conceptual parts.

## Practical Target State

The clean target state is:

- MetalLB continues to own the LAN IP pool
- Traefik owns the shared web ingress IP `192.168.1.50`
- Pi-hole resolves local hostnames to that IP
- web apps move behind Traefik and become `ClusterIP` services where sensible
- non-web protocols stay separate unless there is a deliberate TCP/UDP ingress design

## Short Summary

The intended network model is straightforward:

- one LAN IP for Traefik
- many DNS names in Pi-hole pointing to that one IP
- Traefik routing each hostname to the right internal Kubernetes service

That is the right replacement for the current pattern of exposing many services as `IP:port` on MetalLB addresses.
