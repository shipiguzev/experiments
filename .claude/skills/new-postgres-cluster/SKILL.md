---
name: new-postgres-cluster
description: Scaffold a new independent Zalando postgres-operator cluster (own namespace, own values file), following the postgres2/pg_exporter pattern. Use when the user wants to add another Postgres cluster/namespace for testing (e.g. a different exporter, a different version).
---

Adds a new PostgreSQL cluster as its own Helm release under the shared `postgres-operator`,
isolated in its own namespace, following the existing `postgres` → `postgres2` pattern (see
`values-postgres2-pgexporter.yaml` / `docs/postgres-monitoring-guide-pg-exporter.md`). Ask the user
for: the namespace/cluster-name suffix, the PostgreSQL major version, and which metrics sidecar to
use (`postgresExporter` — the chart default, port 9187 — or `pgExporter` — pgsty's alternative,
port 9630; not both, unless they specifically want to compare the two).

## 1. New values override

Create `charts/postgres-cluster/values-<name>.yaml` (model: `values-postgres2-pgexporter.yaml`):

```yaml
namespace: <name>
operatorNamespace: postgres-operator   # shared operator, same for every cluster

clusterName: postgres-cluster-<name>
teamId: test

postgresql:
  version: "18"          # must be <= the enum the CRD currently allows, see step 2
  volumeSize: 1Gi
  numberOfInstances: 3

databases:
  app: monitoring

sidecars:
  # disable whichever exporter isn't wanted — templates/postgresql.yaml renders each sidecar
  # only `{{- if .Values.sidecars.X }}`, so null actually omits it rather than leaving a default
  postgresExporter: null
  pgExporter:
    image: pgsty/pg_exporter:arm64
    resources:
      requests: {cpu: 50m, memory: 256Mi}
      limits: {cpu: 250m, memory: 512Mi}

podConfig: {}

# Required in EVERY namespace the operator manages, even clusters with no real WAL-G backups:
# pod_environment_secret is a single fixed secret name the operator injects from every managed
# cluster's own namespace — StatefulSet creation fails without it if the namespace has no real
# walg-config Secret already (see templates/secret-walg-config-stub.yaml).
walgSecretStub: true

monitoring:
  namespace: monitoring
  scrapeInterval: 30s
```

## 2. Check the CRD supports the target version

The bundled CRD (`postgresqls.acid.zalan.do`) only allows up to whatever major version was patched
in last (this repo has patched it to include `"18"`, see
`docs/postgres-major-version-upgrade-14-to-18.md` step 2). Confirm before applying:

```bash
kubectl get crd postgresqls.acid.zalan.do \
  -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.postgresql.properties.version.enum}'
```

If the wanted version is missing, patch it — and remember this patch **reverts on every
operator pod restart** (`ensureCRDs` at startup resyncs to the operator's built-in schema), not
just on `helm upgrade`, so re-check after any operator restart:

```bash
kubectl apply -f https://raw.githubusercontent.com/zalando/postgres-operator/master/charts/postgres-operator/crds/postgresqls.yaml
```

## 3. Image prep (only if using `pgExporter`)

`pgsty/pg_exporter` is a multi-platform image — kind can't load it directly by tag. Pull the exact
per-platform digest, retag, then load (see `docs/postgres-monitoring-guide-pg-exporter.md` "Проблема
multi-platform образов в kind" for the full walkthrough):

```bash
docker buildx imagetools inspect pgsty/pg_exporter:latest   # find the arm64/amd64 digest
docker pull pgsty/pg_exporter:latest@sha256:<digest>
docker tag pgsty/pg_exporter:latest@sha256:<digest> pgsty/pg_exporter:arm64
docker save pgsty/pg_exporter:arm64 -o /tmp/pg_exporter.tar
kind load image-archive /tmp/pg_exporter.tar
```

Not needed if using the default `postgresExporter` sidecar (`prometheuscommunity/postgres-exporter`
pulls normally from a registry).

## 4. Install

```bash
kubectl create namespace <name>   # or --create-namespace on the helm command below
helm upgrade --install postgres-cluster-<name> ./charts/postgres-cluster \
  --namespace <name> --create-namespace \
  --values charts/postgres-cluster/values.yaml \
  --values charts/postgres-cluster/values-<name>.yaml
```

Note `values.yaml` (the chart defaults) is passed *and then* the override — `helm upgrade` doesn't
chain values files from previous invocations, and the override alone won't have every base key the
templates expect.

## 5. Monitoring — no extra wiring needed

Unlike the ClickHouse pattern, nothing needs to be added to `charts/monitoring-extras/values.yaml`:
`templates/service-metrics.yaml` and `templates/vmservicescrape.yaml` are rendered per-release from
`.Values.namespace`/`.Values.clusterName`, so the new cluster gets its own headless Service +
`VMServiceScrape` automatically. The existing Postgres dashboards in Grafana already parameterize
by `$namespace`/`$cluster` template variables against the shared VictoriaMetrics datasource — no new
datasource entry is needed (ClickHouse needs one because each ClickHouse cluster is its own HTTP
datasource; Postgres clusters all feed the same VictoriaMetrics instance).

## 6. Verify

- `kubectl get postgresql -n <name> -w` — reaches `Running`.
- `kubectl get pods -n <name>` — each pod `2/2 Running` (note: the `postgres` container has no
  readiness/liveness probe, so `Running` alone doesn't prove Patroni is actually up — see
  `docs/postgres-walg-backup-setup.md` §7.2).
- `kubectl get vmservicescrape -n monitoring <clusterName>` — `operational`.
- Confirm the right exporter is live: `kubectl port-forward <pod> -n <name> 9187:9187` (or
  `9630:9630` for pg_exporter) → `/metrics`.

## 7. Document

Per CLAUDE.md: update README's «Структура репозитория» block to mention the new values file in the
same commit as the file itself. If the cluster exists to test something specific (a different
exporter, a version compat check), say so in a comment in the values file, not just the commit
message — follow the style already in `values-postgres2-pgexporter.yaml`.
