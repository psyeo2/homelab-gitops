Hope this will magically fix it:
```bash
kubectl rollout restart daemonset/longhorn-csi-plugin -n longhorn-system
```
Then watch to see if it does:
```bash
kubectl get pods -n longhorn-system -o wide | grep longhorn-csi-plugin
```