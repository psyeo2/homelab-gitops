# Creating Sealed Secrets

1. Create `secret-plaintext.example.yaml` locally:
   ```bash
   kubectl create secret generic <secret-namespace> \
      --from-literal=<SECRET_NAME>=<secret> \
      --namespace apps \
      --dry-run=client -o yaml > exposed-secret.yaml
   ```
2. Generate a SealedSecret:
   ```bash
   kubeseal --controller-name sealed-secrets-controller --controller-namespace sealed-secrets --format yaml < exposed-secret.yaml > sealedsecret.yaml
   ```
3. Copy output from above to `sealedsecret.yaml` in the repo, then delete files created by previous steps(do not commit plaintext with real key).


kubectl create secret generic <secret-namespace> \
    --from-literal=<SECRET_NAME>=<secret> \
    --namespace apps \
    --dry-run=client -o yaml > exposed-secret.yaml

kubeseal --controller-name sealed-secrets-controller --controller-namespace sealed-secrets --format yaml < exposed-secret.yaml > sealedsecret.yaml
