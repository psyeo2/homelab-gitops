# cloudflare-ddns

Updates Cloudflare DNS records to the cluster egress public IP from inside Kubernetes.

## Configure

1. Edit `secret.yaml`:
   - Set `API_KEY` (same as your previous docker setup).
2. Edit `deployment.yaml` if needed:
   - `ZONE` defaults to `7marketplace.uk`
   - `SUBDOMAIN` defaults to `ddns`

This deployment intentionally mirrors your prior compose env format:
- `API_KEY`
- `ZONE`
- `SUBDOMAIN`
