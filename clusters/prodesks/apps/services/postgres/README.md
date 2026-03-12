# postgres

CloudNativePG operator + starter PostgreSQL cluster for `prodesks`.

## What gets deployed

- `cnpg-operator.yaml`: Argo CD `Application` installing the CloudNativePG Helm chart.
- `cnpg-cluster.yaml`: A 1-instance PostgreSQL cluster named `prodesks-postgres` in namespace `apps`.

## Useful commands

```bash
# Operator status
kubectl -n cnpg-system get pods

# Cluster status
kubectl -n apps get cluster.postgresql.cnpg.io
kubectl -n apps get pods -l cnpg.io/cluster=prodesks-postgres

# Connection services
kubectl -n apps get svc | grep prodesks-postgres
kubectl -n apps get svc prodesks-postgres-rw-lb -o wide

# App user password (created by CNPG)
kubectl -n apps get secret prodesks-postgres-app -o jsonpath='{.data.password}' | base64 -d && echo
```

Connect from LAN once MetalLB assigns an external IP:

```bash
psql "host=<metallb-ip> port=5432 dbname=app user=app sslmode=disable"
```
