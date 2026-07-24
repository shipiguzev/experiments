---
name: new-clickhouse-cluster
description: Scaffold a new ClickHouse test cluster (chi-test-N) following the values-test2/values-test3 pattern, wired into monitoring. Use when the user wants to add another ClickHouse cluster/namespace for testing (e.g. a different version, a different config).
---

Adds a new ClickHouse cluster as its own Helm release, isolated in its own namespace, following the
existing `chi-test` → `chi-test-2` → `chi-test-3` pattern. Ask the user for: the cluster number/name
suffix, and whether it pins a specific ClickHouse image version (like `chi-test-3` does, to test
dashboard compatibility across versions) or tracks whatever `values.yaml`'s default image is.

## 1. New values override

Create `charts/clickhouse-cluster/values-testN.yaml` (model: `values-test3.yaml`):

```yaml
# Nth cluster, separate namespace — install as its own release:
#   helm install chi-test-N ./clickhouse-cluster -f values-testN.yaml
namespace: clickhouse-N
chiName: chi-test-N
clusterName: testN

# only if pinning a version, otherwise omit and it inherits values.yaml's default image:
image: clickhouse/clickhouse-server:X.Y.Z

backup:
  s3:
    pathPrefix: chi-test-N
```

`pathPrefix` must be unique per cluster — it's the S3 prefix `clickhouse-backup` writes under in the
shared bucket.

## 2. Wire it into monitoring

Add an entry to `charts/monitoring-extras/values.yaml` under `clickhouseClusters:`:

```yaml
  - name: chi-test-N
    url: http://clickhouse-chi-test-N.clickhouse-N.svc.cluster.local:8123
    username: grafana_monitoring
    password: monitoring
```

The URL follows `http://clickhouse-<chiName>.<namespace>.svc.cluster.local:8123` — get this wrong
and the datasource silently fails health checks. `grafana_monitoring`/`monitoring` is the existing
scoped monitoring user (see the memory note on its grants) — reuse it, don't create a new user
unless this cluster needs different grants for a reason.

## 3. Install

```bash
helm install chi-test-N ./charts/clickhouse-cluster -f charts/clickhouse-cluster/values-testN.yaml
helm upgrade monitoring-extras ./charts/monitoring-extras \
  --namespace monitoring --values charts/monitoring-extras/values.yaml --reset-values
```

## 4. Verify

- `kubectl get chi -n clickhouse-N` — cluster reaches `Completed`/`Running`.
- New datasource shows up healthy in Grafana (`/api/datasources` + health check), matching the other
  `chi-test*` entries.

## 5. Document

Per CLAUDE.md: update README's «Структура репозитория» block to mention the new values file in the
same commit. If the cluster exists to test something specific (version compat, a config variant —
like `chi-test-3` testing dashboard compatibility with 23.3.2), say so in a comment in the values
file itself, not just the commit message.
