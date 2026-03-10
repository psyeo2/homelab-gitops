# cloudflare-ddns

Updates Cloudflare DNS records to the cluster egress public IP from inside Kubernetes.

## Configure

1. Edit `secret-plaintext.example.yaml` locally:
   - Set `API_KEY` (same as your previous docker setup).
2. Generate a SealedSecret:
   - `kubeseal --controller-name sealed-secrets-controller --controller-namespace sealed-secrets --format yaml < secret-plaintext.example.yaml > sealedsecret.yaml`
3. Commit `sealedsecret.yaml` (do not commit plaintext with real key).
4. Edit `deployment.yaml` if needed:
   - `ZONE` defaults to `7marketplace.uk`
   - `SUBDOMAIN` defaults to `ddns`

This deployment intentionally mirrors your prior compose env format:
- `API_KEY`
- `ZONE`
- `SUBDOMAIN`
