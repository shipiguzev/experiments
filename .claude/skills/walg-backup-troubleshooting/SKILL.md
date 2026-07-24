---
name: walg-backup-troubleshooting
description: Diagnose WAL-G backup/archiving problems on the Zalando postgres-operator cluster — archive_command silently stuck on /bin/true, replicas failing after a major upgrade or repeated cluster teardown, or the operator not picking up new WAL-G config. Use when base backups or continuous WAL archiving aren't working as expected.
---

Checklist for the two WAL-G failure modes this repo has actually hit, both silent (pods start
cleanly, no obvious error) rather than crashing loudly. Full setup reference:
`docs/postgres-walg-backup-setup.md`. See also `postgres-major-version-upgrade` for the upgrade-
specific variant of the stale-archive bug.

## First: confirm which symptom you actually have

```bash
kubectl exec -n <ns> <primary-pod> -c postgres -- psql -U postgres -c "show archive_command;"
```

- `envdir "/run/etc/wal-e.d/env" wal-g wal-push "%p"` → archiving is configured; if backups still
  seem missing, check bucket contents (step 3 below) rather than config.
- `/bin/true` → go to **A. archive_command stuck on /bin/true**.

If archiving looks fine but replicas are crash-looping after an upgrade or cluster
recreate/teardown cycle → go to **B. stale timeline history**.

## A. `archive_command` is `/bin/true` (WAL never archived)

**Root cause**: Spilo's `configure_spilo.py` decides `USE_WALE`/`archive_command` at bootstrap
purely from **environment variables** (`WAL_S3_BUCKET`/`WALE_S3_PREFIX`/`WALG_S3_PREFIX`, or the
GS/Azure equivalents) — never from files under a mounted secret path. So:

- **`configAwsOrGcp.additional_secret_mount`/`additional_secret_mount_path` alone is not enough.**
  It gives `wal-g` credentials for *manual* invocation (`envdir` reads the mounted files fine when
  you call `wal-g backup-push` by hand — which is exactly why this bug hides during setup
  verification), but `configure_spilo.py` never sees those files, so `USE_WALE` stays false and
  `archive_command` hardcodes to `/bin/true`. The pod starts with zero errors.
- **Don't "fix" it by also adding `wal_s3_bucket`** to get `WAL_S3_BUCKET` into the env. With
  `additional_secret_mount` still active, `configure_spilo.py`'s `write_wale_environment()` tries to
  write files into the *same* directory the secret already occupies read-only
  (`/run/etc/wal-e.d/env`) → crashes with `OSError: [Errno 30] Read-only file system`, and Patroni
  never starts at all (see step "Patroni didn't start" below).

**Fix**: use `configKubernetes.pod_environment_secret`, not a mounted volume. This injects the
secret's keys as real container env vars (`envFrom`), which `configure_spilo.py` can see at
bootstrap — and it creates `/run/etc/wal-e.d/env` itself (the directory isn't occupied by a volume
in this scheme), so `additional_secret_mount` isn't needed at all:

```yaml
# postgres/operator/postgres-operator-values.yaml
configKubernetes:
  pod_environment_configmap: "postgres-operator/postgres-pod-config"
  pod_environment_secret: "walg-config"   # secret lives in the CLUSTER's own namespace, not postgres-operator
```

```bash
helm upgrade postgres-operator postgres-operator-charts/postgres-operator \
  --namespace postgres-operator --values postgres/operator/postgres-operator-values.yaml
kubectl rollout restart deployment postgres-operator -n postgres-operator
```

The forced restart matters: right after changing operator config, the operator sometimes computes
the correct diff (`new statefulset containers's postgres environment does not match...`) but
applies it incorrectly once — a full operator restart forces a clean resync of every managed
cluster.

Re-verify: `archive_command` should now show the real `wal-g wal-push` command, and:

```bash
kubectl exec -n <ns> <primary-pod> -c postgres -- ls -la /run/etc/wal-e.d/env/
```

should show **regular files** (Spilo-generated), not symlinks (which is what a mounted-secret
volume would produce instead).

### If Patroni didn't start at all (the crash variant of this same mistake)

```bash
kubectl exec -it <pod> -n <ns> -- bash -c "patronictl list"
# WARNING - Listing members: No cluster names were provided
```

This means `/run/postgres.yml` was never generated — Patroni itself never started. Check:

```bash
kubectl logs -n <ns> <pod> -c postgres --tail=50
# OSError: [Errno 30] Read-only file system: '/run/etc/wal-e.d/env/WALE_S3_PREFIX'
```

Fix: remove `wal_s3_bucket` from the operator config (keep only `pod_environment_secret`), then
`helm upgrade` again. If Patroni was already dead when the operator restarted, the operator's normal
rolling update **won't recover it either** — it coordinates restarts via the Patroni REST API
(`:8008/patroni`), and with Patroni down that just logs `connection refused` /
`postpone pod recreation until next sync` forever. Once the StatefulSet itself is confirmed correct
(env vars present, see below), delete pods manually — replicas first, leader last:

```bash
kubectl delete pod <replica-1> <replica-2> -n <ns>
kubectl wait --for=condition=Ready pod <replica-1> <replica-2> -n <ns> --timeout=120s
kubectl delete pod <leader> -n <ns>   # last — Patroni fails over to an already-recreated replica
```

Safe because `PGDATA` lives on a PVC — deleting the pod doesn't touch the data.

### StatefulSet sanity check (env actually present)

```bash
kubectl get statefulset <cluster> -n <ns> -o json | python3 -c "
import json, sys
d = json.load(sys.stdin)
c = [c for c in d['spec']['template']['spec']['containers'] if c['name']=='postgres'][0]
print([e['name'] for e in c.get('env', []) if e.get('valueFrom', {}).get('secretKeyRef', {}).get('name') == 'walg-config'])"
```

Expect `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT`, `WALG_S3_PREFIX`, etc. Note the
`postgres` container has no readiness/liveness probe — `2/2 Running` proves only that the container
processes are alive, not that Patroni/Postgres actually started; rely on this check plus the
`archive_command` check above, not pod status.

## B. Stale timeline history (replicas fail after upgrade or repeated teardown)

**Symptom**: replicas `start failed` with `FATAL: requested timeline N is not a child of this
server's history`, and it recurs even after Patroni fully reinitializes the replica from a fresh
basebackup.

**Root cause**: `WALG_S3_PREFIX` is a fixed prefix, not scoped to a particular cluster incarnation
or major version. Old WAL/`.history` files from earlier test runs, prior major-version upgrades, or
past leader/replica switches accumulate in the same S3 prefix. A brand-new basebackup on a fresh
timeline still hits the archive on recovery, finds a stale higher-numbered `.history` file, and
tries to recover along that unrelated lineage — hence the FATAL repeats even right after a reinit.

This is the same underlying issue whether it's triggered by a `pg_upgrade` (see
`postgres-major-version-upgrade`) or by repeatedly tearing down and recreating the same kind test
cluster without ever clearing its backup prefix. A real, non-test cluster upgraded exactly once
wouldn't hit this — there'd be no leftover incarnation's WAL sitting in the bucket.

**Confirm**, don't assume — check timelines (leading 8 hex digits of `wal_file_name`):

```bash
kubectl exec -n <ns> <leader-pod> -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env wal-g backup-list --detail"
```

Multiple old/high timelines mixed with a much lower fresh one is the signature.

**Fix** — scoped to this cluster's own prefix only, doesn't touch other clusters/buckets:

```bash
kubectl exec -n <ns> <leader-pod> -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env wal-g delete everything --confirm"
kubectl exec -n <ns> <leader-pod> -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env wal-g backup-list"   # confirm empty
kubectl exec -n <ns> <leader-pod> -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env su postgres -c '/scripts/postgres_backup.sh /home/postgres/pgdata/pgroot/data'"
```

Use `/scripts/postgres_backup.sh` (Spilo's own wrapper, runs as `postgres`), not a bare
`wal-g backup-push` — direct calls connect as `root` and fail `password authentication failed`.
Don't manually reinitialize replicas afterward — Patroni's own retry loop (every ~10-20s) recovers
them once the archive stops serving stale history.

## Quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `archive_command` = `/bin/true`, no errors anywhere | Creds only file-mounted (`additional_secret_mount`), never in env | `pod_environment_secret`, not a mounted volume |
| `OSError: Read-only file system` on `WALE_S3_PREFIX`, Patroni never starts | `wal_s3_bucket` set *and* `additional_secret_mount` both targeting the same dir | Drop `wal_s3_bucket`, keep only `pod_environment_secret` |
| `patronictl list` → "No cluster names were provided" | Patroni process never started (see above) | Fix root cause, then manually recreate pods if operator can't reach Patroni API |
| Replicas `start failed`, `FATAL: requested timeline N is not a child` | Stale `.history`/WAL from earlier incarnation in the shared S3 prefix | `wal-g delete everything --confirm` + fresh basebackup via `postgres_backup.sh` |
| `wal-g backup-push` → `password authentication failed for user "root"` | Called directly instead of via the Spilo wrapper | Use `/scripts/postgres_backup.sh`, or set `PGUSER`/`PGHOST` explicitly for manual testing |
