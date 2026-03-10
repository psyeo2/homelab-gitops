# wireguard

Managed via Argo CD `Application`:
- `wireguard.yaml`

Before first real use:
- Set `WG_HOST` in `wireguard.yaml` to your public IP or DNS.
- Forward UDP `51820` on your router to `192.168.1.52`.
