# redis

Redis is deployed as an Argo CD `Application` (`application.yaml`) using the Bitnami `redis` Helm chart.

Current defaults:

- Single-node deployment (`architecture: standalone`)
- Internal-only service (`ClusterIP`)
- Longhorn-backed persistence (`8Gi`)
- Password auth enabled via the sealed secret `redis`

## Credentials

- The Bitnami chart reads the password from the unsealed Secret created by `sealedsecret.yaml`.
- Expected Secret name: `redis`
- Expected key: `REDIS_DEFAULT_PASS`

Username is not used by Redis auth.
