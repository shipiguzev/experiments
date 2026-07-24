---
name: postgres-major-version-upgrade
description: Run an in-place PostgreSQL major version upgrade on the Zalando postgres-operator cluster (Spilo's inplace_upgrade.py), and recover if replicas crash-loop afterward. Use when the user wants to bump postgresql.version on a cluster that already has data, or is debugging a stuck/failed upgrade.
---

Walks a Zalando postgres-operator cluster through Spilo's built-in in-place major version upgrade
(`inplace_upgrade.py`), with real downtime (~10-20s), and the two upgrade-specific failure modes
this repo has actually hit. Full reference: `docs/postgres-major-version-upgrade-14-to-18.md`.

**Ask the user first**: current version, target version, and whether this is the shared "postgres"
namespace or one of the secondary clusters (`new-postgres-cluster` pattern) — the CRD-enum and
image-tag steps below apply regardless, but confirm before running the actual upgrade script since
it is **not reversible via dry-run** (see step 4).

## 1. Bump the operator's image + upgrade-mode config

```yaml
# postgres/operator/postgres-operator-values.yaml
configGeneral:
  docker_image: ghcr.io/zalando/spilo-<target-major>:<tag>   # one Spilo image tag = one PG major

configMajorVersionUpgrade:
  major_version_upgrade_mode: "manual"
  minimal_major_version: "<current-major>"
  target_major_version: "<target-major>"
```

`major_version_upgrade_mode` must live in `configMajorVersionUpgrade`, not `configGeneral` — putting
it there fails `helm upgrade` with `field not declared in schema`.

```bash
helm upgrade postgres-operator postgres-operator-charts/postgres-operator \
  --namespace postgres-operator \
  --values postgres/operator/postgres-operator-values.yaml
```

## 2. Patch the CRD to allow the target version

The chart-bundled CRD only allows versions up to what shipped with the operator (this repo needed
a manual patch to unlock `"18"`):

```bash
kubectl apply -f https://raw.githubusercontent.com/zalando/postgres-operator/master/charts/postgres-operator/crds/postgresqls.yaml
kubectl get crd postgresqls.acid.zalan.do \
  -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.postgresql.properties.version.enum}'
```

**This patch does not stick.** The operator's `ensureCRDs` resyncs to its built-in schema on every
pod (re)start — not only after `helm upgrade`, but after ANY restart of the `postgres-operator`
deployment, including `kubectl rollout restart` used elsewhere in this repo's troubleshooting.
Re-check the enum and re-apply after any such restart, or the next manifest apply with the new
version fails with `Unsupported value`.

## 3. Bump `postgresql.version` in the cluster's values, apply

```yaml
postgresql:
  version: "<target-major>"   # was "<current-major>"
```

```bash
helm upgrade <release> ./charts/postgres-cluster \
  --namespace <ns> \
  --values charts/postgres-cluster/values.yaml \
  --values <every values-step*/override file this cluster already uses, in order>
```

`helm upgrade` doesn't remember `-f` from earlier invocations — omitting an earlier override file
silently reverts those values to chart defaults. This step only recreates pods on the new Spilo
image; `SELECT version()` still reports the old major version afterward — that's expected, the real
data upgrade happens in step 5.

## 4. Find the Leader

```bash
kubectl exec -it <pod-0> -n <ns> -c postgres -- bash -c "patronictl -c /run/postgres.yml list"
```

The upgrade only runs on the Leader; running it on a Replica fails with `PostgreSQL is not running
or in recovery`.

## 5. Run the upgrade — this is NOT a dry run

**`inplace_upgrade.py` has no safe check-only mode.** It always runs `pg_upgrade --check` first,
and if the clusters are compatible, it immediately continues into the real data upgrade in the same
invocation, without another confirmation prompt. There is no way to see just the compatibility
report without also performing the upgrade if it passes.

```bash
kubectl exec -it <leader-pod> -n <ns> -c postgres -- \
  bash -c "su postgres -c 'python3 /scripts/inplace_upgrade.py <total-pod-count> 2>&1'"
```

The argument is the **total number of pods**, not replica count — e.g. `3` for a 1-leader/2-replica
cluster (`3` replicas would error `number of replicas does not match (3 != 2)`). Must run via
`su postgres -c`, not as root (`initdb: error: cannot be run as root`). Takes ~15-20s total,
including ~9s of actual downtime.

## 6. Verify data + replication

```bash
kubectl exec -it <leader-pod> -n <ns> -- psql -U postgres -c "SELECT version();"   # new major version
kubectl exec -it <pod-0> -n <ns> -c postgres -- bash -c "patronictl -c /run/postgres.yml list"
```

If `patronictl list` shows `start failed` on one or both replicas instead of `streaming` — do not
treat the upgrade as done — go to step 7.

## 7. If replicas crash-loop with `start failed` after the upgrade

**Symptom** — repeats even after Patroni auto-reinitializes the replica from a fresh basebackup:

```
FATAL:  requested timeline N is not a child of this server's history
```
or, in `configure_spilo`/WAL-G's own log lines instead:
```
ERROR: Archive '00000005.history' does not exist.
```

Check the actual postgres log inside the pod if `kubectl logs` doesn't show it (Spilo routes
postmaster output to `csvlog`, not container stdout):

```bash
kubectl exec -n <ns> <pod> -c postgres -- \
  bash -c "tail -n 60 /home/postgres/pgdata/pgroot/pg_log/postgresql-*.log"
```

**Root cause**: `WALG_S3_PREFIX` is not scoped per cluster incarnation — leftover WAL/`.history`
files from earlier timelines (previous test runs, prior leader/replica switches) sit in the same S3
prefix. `pg_upgrade` correctly starts the new data dir on a fresh timeline 1, but Patroni's
`recovery_target_timeline: latest` finds the stale higher-numbered `.history` file and tries to
recover along that unrelated lineage. Confirm by checking timelines (leading 8 hex digits of
`wal_file_name`):

```bash
kubectl exec -n <ns> <leader-pod> -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env wal-g backup-list --detail"
```

Mixed old high timelines next to a fresh low-numbered one on the new major version is the
signature. **Fix** (scoped to this cluster's own `WALG_S3_PREFIX` — safe, doesn't touch other
clusters' backups):

```bash
kubectl exec -n <ns> <leader-pod> -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env wal-g delete everything --confirm"
kubectl exec -n <ns> <leader-pod> -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env su postgres -c '/scripts/postgres_backup.sh /home/postgres/pgdata/pgroot/data'"
```

Use the Spilo-provided `/scripts/postgres_backup.sh`, not a bare `wal-g backup-push` — a direct call
connects as `root` and fails `password authentication failed`. Don't manually reinitialize the
replicas — once the archive stops serving the stale history, Patroni's own retry loop
(~every 10-20s) recovers them on its own.

## Known gotchas (quick table)

| Symptom | Fix |
|---|---|
| CRD enum reverts to old max version | Re-apply the CRD patch after every operator pod restart, not just `helm upgrade` |
| `field not declared in schema` on operator upgrade | Move `major_version_upgrade_mode` etc. into `configMajorVersionUpgrade` |
| `initdb: error: cannot be run as root` | Run via `su postgres -c '...'` |
| `PostgreSQL is not running or in recovery` | Target the Leader, found via `patronictl list` |
| `number of replicas does not match` | Pass total pod count, not replica count |
| Version still reports old major after `helm upgrade` | Expected — the real upgrade is step 5, not the values bump |
| Replicas `start failed`, `FATAL: requested timeline N is not a child` | Stale WAL-G archive — `wal-g delete everything --confirm` + fresh basebackup (step 7), not a reinit |
