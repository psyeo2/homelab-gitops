# cloudflare-ddns

Updates Cloudflare DNS records to the cluster egress public IP from inside Kubernetes.

## Configure

1. Create `secret-plaintext.example.yaml` locally:
   ```bash
   kubectl create secret generic cloudflare-ddns \
      --from-literal=API_KEY=<my_api_key> \
      --namespace apps \
      --dry-run=client -o yaml > cloudflare-secret.yaml
   ```
2. Generate a SealedSecret:
   ```bash
   kubeseal --controller-name sealed-secrets-controller --controller-namespace sealed-secrets --format yaml < cloudflare-secret.yaml > sealedsecret.yaml
   ```
3. Copy output from above to `sealedsecret.yaml` in the repo, then delete files created by previous steps(do not commit plaintext with real key).
4. Edit `deployment.yaml` if needed:
   - `ZONE` defaults to `7marketplace.uk`
   - `SUBDOMAIN` defaults to `ddns`
