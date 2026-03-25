# vaultwarden

Vaultwarden is deployed as an Argo CD `Application` (`application.yaml`) using the `guerzon/vaultwarden` Helm chart.

Current defaults:

- Single replica, pinned to Longhorn-backed persistent storage.
- Internal-only service (`ClusterIP`) until you decide the real ingress/tunnel/LAN exposure path.
- `signupsAllowed: true` for bootstrap. After creating your first account and wiring an admin token, flip this to `false`.

## Before first real use

1. Decide the public URL and set `domain` in `application.yaml`.
2. Generate an admin token secret, seal it, and then uncomment the `adminToken` block in `application.yaml`.
3. If you expose Vaultwarden to official Bitwarden clients or browser extensions, use HTTPS.

## Admin token

Create a plaintext secret locally from `secret-plaintext.example.yaml`, then seal it:

```bash
kubeseal --controller-name sealed-secrets-controller --controller-namespace sealed-secrets --format yaml < secret-plaintext.example.yaml > sealedsecret.yaml
```

Then:

- Add `sealedsecret.yaml` to `kustomization.yaml`.
- Uncomment the `adminToken` block in `application.yaml`.
- Delete the local plaintext secret file.

Plain tokens work, but Vaultwarden prefers an Argon2 PHC string for `ADMIN_TOKEN`.

## Exposure options

- For LAN-only access, change `service.type` to `LoadBalancer` and pin a free MetalLB IP.
- For normal Bitwarden clients, prefer an HTTPS hostname via Traefik or Cloudflare Tunnel and set `domain` to match it exactly.
