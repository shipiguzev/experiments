---
name: new-cnpg-cluster
description: Scaffold a new CloudNativePG (CNPG) cluster release — own namespace, own values file, own backup prefix — alongside the existing cnpg-cluster. Use when the user wants another CNPG-managed Postgres cluster in the same kind stand.
---

Adds another CloudNativePG-managed cluster as its own Helm release of the `charts/cnpg-cluster`
chart, sharing the single `cnpg-operator` already installed (CNPG's operator is `clusterWide: true`
by default — it watches every namespace, no per-cluster operator config needed). Ask the user for:
the namespace/cluster-name suffix, the PostgreSQL major version/image, and the backup schedule.

## 1. New values override

Create `charts/cnpg-cluster/values-<name>.yaml` (model: `values.yaml`, which is itself modeled on
`charts/postgres-cluster/values-production.yaml` for comparable-workload parity between operators):

```yaml
namespace: <name>
clusterName: cnpg-cluster-<name>

postgresql:
  imageName: ghcr.io/cloudnative-pg/postgresql:18   # floating major-version tag, always latest patch
  instances: 3
  storageSize: 1Gi

backup:
  endpointURL: http://minio.minio.svc.cluster.local:9000
  # own prefix inside the SAME bucket other clusters already use — must not collide with
  # cnpg/cnpg-cluster or spilo/postgres-cluster:
  destinationPath: s3://postgres-backups/cnpg/cnpg-cluster-<name>
  credentialsSecretName: cnpg-<name>-minio-creds
  # CNPG's cron has a leading seconds field (6 fields, robfig/cron) — "0 0 2 * * *" == daily 02:00,
  # NOT the 5-field cron Zalando/WAL-G uses. Getting this wrong either fails CRD validation or
  # silently shifts the schedule.
  schedule: "0 0 2 * * *"
  retentionPolicy: "7d"

monitoring:
  namespace: monitoring
  scrapeInterval: 30s
```

`destinationPath` and `credentialsSecretName` must both be unique per cluster — they're what keeps
this cluster's backups from colliding with the existing `cnpg-cluster`'s in the shared bucket.

## 2. Bucket + creds secret (manual, not templated)

Same reasoning as the existing cluster and as Zalando's `walg-config`: real S3 credentials never go
into a values file in git, so the Secret is created by hand, in the new cluster's own namespace:

```bash
kubectl create namespace <name>
MINIO_POD=$(kubectl get pods -n minio -l app=minio -o jsonpath="{.items[0].metadata.name}")
kubectl exec -n minio "$MINIO_POD" -- mc mb local/postgres-backups   # no-op if bucket already exists

kubectl create secret generic cnpg-<name>-minio-creds -n <name> \
  --from-literal=ACCESS_KEY_ID=minioadmin \
  --from-literal=ACCESS_SECRET_KEY=minioadmin
kubectl label secret cnpg-<name>-minio-creds -n <name> cnpg.io/reload=true
```

The key names `ACCESS_KEY_ID`/`ACCESS_SECRET_KEY` are fixed — that's what `s3Credentials` in the
`Cluster` CR expects (see `templates/cluster.yaml`), not the `AWS_ACCESS_KEY_ID`-style names WAL-G
uses. The `cnpg.io/reload=true` label makes the operator restart the cluster if the secret's
contents change later, not just on first creation.

## 3. Install

```bash
helm upgrade --install cnpg-cluster-<name> ./charts/cnpg-cluster \
  --namespace <name> \
  --values charts/cnpg-cluster/values-<name>.yaml
```

Expect the operator's own deprecation warning about `barmanObjectStore` (native backup support) —
this is normal and shared with the existing cluster; see the "Известные особенности" in
`docs/cnpg-cluster-setup.md` for why this repo hasn't migrated to the Barman Cloud Plugin.

Bootstrap (initdb on primary → 2 replicas join) takes ~1.5–2 minutes.

## 4. Monitoring — wired automatically per release

`templates/vmpodscrape.yaml` renders a `VMPodScrape` from `.Values.namespace`/`.Values.clusterName`
already, so the new cluster gets scraped without touching `monitoring-extras` — CNPG instances
expose metrics directly on the pod (`:9187`, no headless Service, unlike
Zalando/ClickHouse's `VMServiceScrape` pattern). The operator-level `VMPodScrape`
(`charts/monitoring-extras/templates/vmpodscrape-cnpg-operator.yaml`) is shared across every CNPG
cluster and needs no change. The existing `CloudNativePG` dashboard in Grafana is parameterized by
`$namespace`/`$cluster` variables — no new datasource or dashboard needed.

## 5. Verify

```bash
kubectl get cluster -n <name> -w
# NAME                 INSTANCES  READY  STATUS                       PRIMARY
# cnpg-cluster-<name>  3          3      Cluster in healthy state     cnpg-cluster-<name>-1

kubectl get vmpodscrape -n monitoring   # new entry present, operational
```

Manual backup smoke-test, then confirm objects land under the *new* prefix, not the existing
cluster's:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: cnpg-cluster-<name>-manual-1
  namespace: <name>
spec:
  cluster:
    name: cnpg-cluster-<name>
  method: barmanObjectStore
EOF
kubectl get backup cnpg-cluster-<name>-manual-1 -n <name> -w   # phase: completed

kubectl exec -n minio "$MINIO_POD" -- mc ls --recursive local/postgres-backups/cnpg/cnpg-cluster-<name>/
```

## 6. Document

Per CLAUDE.md: update README's «Структура репозитория» block in the same commit as the new values
file. State the reason for the new cluster (a version comparison, a config variant) as a comment in
the values file itself, matching the style already in `values.yaml`/`cnpg-cluster-setup.md`.
