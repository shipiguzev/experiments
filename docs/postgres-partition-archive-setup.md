# Архивирование старых партиций pg_partman в S3

Инструкция описывает `CronJob`, который для каждой управляемой `pg_partman` таблицы находит партиции старше заданного retention, выгружает их (`pg_dump -Fc`) в отдельный S3-бакет и только после подтверждённой успешной загрузки отсоединяет их от родительской таблицы (`DETACH PARTITION`) — **без удаления** самой партиции. Удаление отсоединённых таблиц — сознательно отдельная задача, не решаемая этой инструкцией.

## Область применения

Предполагается, что уже развёрнуты:

- `postgres-operator`, кластер `postgres-cluster` в namespace `postgres`, версия 18;
- `pg_partman` и партиционированные таблицы (`public.events_daily`, `public.metrics_daily`, `public.request_logs_daily`) — см. [`docs/postgres-pg-partman-setup.md`](postgres-pg-partman-setup.md);
- MinIO (`docs/postgres-walg-backup-setup.md`, шаг 1) — используется тот же инстанс, но **отдельный** бакет, не тот, что для WAL-G.

## Структура файлов

```
experiments/
└── charts/
    ├── postgres-cluster/
    │   ├── templates/
    │   │   ├── cronjob-partition-archive.yaml
    │   │   ├── configmap-partition-archive-script.yaml
    │   │   └── configmap-exporter-queries.yaml          # + pg_partman_partitions custom query
    │   └── values-step8-partition-archive.yaml
    └── monitoring-extras/
        └── dashboards/
            └── postgresql-cluster-overview.json          # + секция "Partitioning (pg_partman)"
```

## 1. Как это работает

Вместо того чтобы самостоятельно парсить имена партиций (`<parent>_pYYYYMMDD`) регэкспом по `pg_inherits` — что хрупко и совпадает только для суточного интервала, — скрипт целиком опирается на служебные объекты самого `pg_partman`:

| Объект | Зачем |
|---|---|
| `partman.part_config` | Единственный источник правды о том, какие таблицы вообще под управлением `pg_partman` — список таблиц в скрипте не хардкожен, добавление/удаление партиционированной таблицы подхватывается на следующий запуск само |
| `partman.show_partitions(parent, 'ASC')` | Список текущих дочерних партиций — не нужно самому ходить в `pg_inherits` |
| `partman.show_partition_info(child, interval, parent)` | Реальные границы партиции (`child_start_time`) — из констрейнта, а не из имени таблицы; для id/serial-партиционирования возвращает `NULL`, такие партиции скрипт пропускает |
| `partman.drop_partition_time(parent, p_retention, p_keep_table => true)` | Официальная функция `pg_partman` для вывода партиций из retention — с `p_keep_table => true` она **только детачит**, `DROP TABLE` не вызывается вообще (проверено по исходнику функции, см. раздел 3) |

Критерий отбора в скрипте (`(child_start_time + partition_interval) < (now() - retention)`) — это **тот же самый** критерий, который использует сам `drop_partition_time` внутри. Поэтому набор партиций, которые скрипт выгружает в S3, гарантированно совпадает с тем, что `drop_partition_time` затем отсоединит.

Порядок действий на каждый управляемый `parent_table`:
1. Найти партиции старше retention (см. критерий выше).
2. Если таких нет — перейти к следующей таблице.
3. Для каждой такой партиции: `pg_dump -Fc -t schema.partition | mc pipe archive/<bucket>/<db>/<schema>/<partition>.dump`, затем `mc stat` — убедиться, что объект реально появился в S3 и его размер не нулевой.
4. Только если **все** партиции этой таблицы выгружены успешно — один раз вызвать `partman.drop_partition_time(parent, p_retention => retention, p_keep_table => true)`. Если хотя бы одна выгрузка упала — детач для этой таблицы в этом запуске пропускается целиком (не частично), job завершается с ненулевым кодом.

`part_config.retention` при этом остаётся `NULL` — retention передаётся explicit-параметром `p_retention` в каждый вызов `drop_partition_time`, а не хранится в конфиге. Это осознанное решение: `pg_partman_bgw` (см. [`docs/postgres-pg-partman-setup.md`](postgres-pg-partman-setup.md), шаг 4) продолжает раз в час вызывать `run_maintenance_proc()`, но раз `retention` в `part_config` не задан — сам он никакие партиции не трогает. Единственный, кто управляет детачем — этот `CronJob`, и только после подтверждённой выгрузки в S3.

## 2. Ручные шаги перед первым запуском

### 2.1. Бакет в MinIO

Отдельный от `postgres-backups` (WAL-G) бакет — разные политики хранения/доступа для PITR-бэкапов и архивов партиций:

```bash
MINIO_POD=$(kubectl get pods -n minio -l app=minio -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n minio "$MINIO_POD" -- mc alias set local http://localhost:9000 minioadmin minioadmin
kubectl exec -n minio "$MINIO_POD" -- mc mb local/partition-archive
```

### 2.2. Секрет с доступом к S3

Тот же принцип, что и `walg-config` (см. [`docs/postgres-walg-backup-setup.md`](postgres-walg-backup-setup.md), шаг 3) — секрет с credentials создаётся вручную, не хранится в values-файле:

```bash
kubectl create secret generic partition-archive-s3 \
  --namespace postgres \
  --from-literal=AWS_ACCESS_KEY_ID=minioadmin \
  --from-literal=AWS_SECRET_ACCESS_KEY=minioadmin \
  --from-literal=AWS_ENDPOINT=http://minio.minio.svc.cluster.local:9000
```

> Для реального S3 — отдельный IAM-пользователь с правом записи только в этот бакет, ключи вместо `minioadmin`/`minioadmin`.

## 3. Values и деплой

`charts/postgres-cluster/values-step8-partition-archive.yaml`:

```yaml
partitionArchive:
  enabled: true
  schedule: "0 3 * * *"
  retention: "30 days"
  database: app
  credentialsSecret: postgres.postgres-cluster.credentials.postgresql.acid.zalan.do
  image: ghcr.io/zalando/spilo-18:4.1-p1
  mcImage: minio/mc:latest
  s3:
    endpoint: http://minio.minio.svc.cluster.local:9000
    bucket: partition-archive
    secretName: partition-archive-s3
```

Применяется поверх всей цепочки предыдущих шагов:

```bash
helm upgrade postgres-cluster ./charts/postgres-cluster \
  --namespace postgres \
  --values charts/postgres-cluster/values.yaml \
  --values charts/postgres-cluster/values-step3-walg.yaml \
  --values charts/postgres-cluster/values-step4-v18.yaml \
  --values charts/postgres-cluster/values-step6-partman.yaml \
  --values charts/postgres-cluster/values-step7-walg-exporter.yaml \
  --values charts/postgres-cluster/values-step8-partition-archive.yaml
```

### Почему `credentialsSecret` указывает на `postgres`, а не на `app_owner`

`app_owner` — владелец партиционированных таблиц, но это NOLOGIN-роль (проверено на живом кластере: `SELECT rolcanlogin FROM pg_authid WHERE rolname = 'app_owner'` → `f`) — под ней нельзя открыть подключение. Оператор Zalando создаёт Secret с credentials только для ролей, способных логиниться (в этом кластере это `postgres`, `standby`, `monitoring` — `kubectl get secrets -n postgres | grep credentials`). Поэтому job подключается под `postgres` (тем же суперпользователем, под которым во всех остальных доках этого репозитория выполняются ручные `psql`/`pg_dump` команды) — для dev-стенда это осознанное упрощение; в проде для этой задачи стоит завести отдельную LOGIN-роль с членством в `app_owner` и `SET ROLE` перед DDL, а не использовать суперпользователя.

### Почему `PGSSLMODE=require`

Job подключается по TCP к Service кластера, а не через unix-сокет (как sidecar'ы `pg-exporter`/`walg-exporter` внутри того же пода). `pg_hba.conf` на этом кластере (генерируется Patroni, не редактируется вручную):

```
hostnossl all  all  all  reject
hostssl   all  all  all  md5
```

Любое TCP-подключение без SSL отклоняется — без `PGSSLMODE=require` в env контейнера `archive` job не сможет подключиться вообще.

## 4. Проверка

```bash
# CronJob создан, расписание — 03:00 каждый день
kubectl get cronjob postgres-cluster-partition-archive -n postgres

# Разовый ручной запуск вне расписания
kubectl create job -n postgres partition-archive-manual --from=cronjob/postgres-cluster-partition-archive
kubectl wait --for=condition=complete --timeout=90s job/partition-archive-manual -n postgres
kubectl logs -n postgres job/partition-archive-manual

# Содержимое бакета
MINIO_POD=$(kubectl get pods -n minio -l app=minio -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n minio "$MINIO_POD" -- mc ls --recursive local/partition-archive

# Партиция реально отсоединена от parent...
kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app \
  -c "SELECT partition_tablename FROM partman.show_partitions('public.events_daily','ASC') LIMIT 5;"

# ...но данные никуда не делись — таблица осталась как самостоятельная
kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app \
  -c "SELECT count(*) FROM public.events_daily_p<YYYYMMDD>;"

kubectl delete job partition-archive-manual -n postgres
```

Ожидаемый вывод лога успешного запуска (пример — реально прогнано на локальном стенде):

```
Added `archive` successfully.
== public.events_daily: retention 30 days ==
nothing to archive
== public.metrics_daily: retention 30 days ==
-> dumping public.metrics_daily_p20260709 to s3://partition-archive/app/public/metrics_daily_p20260709.dump
-> detaching 1 archived partition(s) from public.metrics_daily (retention_keep_table=true, no DROP)
```

## 5. Логи

Логи процесса архивирования есть, но только как обычные логи контейнера `archive` пода Job'а — в этом стенде нет отдельного стека агрегации логов (Loki/ELK и т.п., есть только VictoriaMetrics для метрик), поэтому доступны они ровно до тех пор, пока существует сам объект `Job`/`Pod`.

```bash
# Список запусков, сохранённых Kubernetes (лидер CronJob'а сам чистит старые Job'ы сверх лимита)
kubectl get jobs -n postgres -l job-name

# Логи конкретного запуска (по имени Job'а, содержит суффикс-timestamp)
kubectl logs -n postgres job/postgres-cluster-partition-archive-<suffix>

# Логи только упавших запусков за последнее время
kubectl get jobs -n postgres -o json \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print([j['metadata']['name'] for j in d['items'] if j['status'].get('failed')])"
```

По умолчанию Kubernetes хранит объекты только 3 последних успешных и 1 последнего неудачного запуска (`successfulJobsHistoryLimit`/`failedJobsHistoryLimit`) — этого мало для расследования проблемы, обнаруженной через день-два после сбоя. В `values-step8-partition-archive.yaml` оба лимита подняты до `5`:

```yaml
partitionArchive:
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
```

Это не централизованное логирование, а просто более долгое хранение стандартных объектов `Job`/`Pod` — если нужна история старше нескольких дней или полнотекстовый поиск по логам, единственный вариант — разворачивать отдельный log-стек (например, Loki, уже совместимый с этим VictoriaMetrics+Grafana стендом через `vmagent`/promtail-аналог), что не входит в эту инструкцию.

## 6. Мониторинг в Grafana

Дашборд `PostgreSQL Cluster Overview` (`charts/monitoring-extras/dashboards/postgresql-cluster-overview.json`) дополнен секцией **Partitioning (pg_partman)** из 4 панелей:

| Панель | Тип | Запрос | Источник |
|---|---|---|---|
| Attached Partitions | timeseries | `pg_partman_partitions_partition_count{namespace=~"$namespace"}` | новая custom-query метрика `postgres-exporter` |
| Detached Partitions (last 24h) | stat | `sum(clamp_min(-delta(pg_partman_partitions_partition_count{...}[24h]), 0)) by (parent_table)` | то же |
| Archive CronJob: Time Since Last Scheduled Run | stat | `time() - max(kube_cronjob_status_last_schedule_time{cronjob=~".*partition-archive.*"}) by (cronjob)` | `kube-state-metrics` (уже в составе `victoria-metrics-k8s-stack`) |
| Archive CronJob: Failed Runs (last 2d) | stat | `sum(max_over_time(kube_job_status_failed{job_name=~".*partition-archive.*"}[2d]))` | то же |

### Метрика количества партиций

Добавлена своя custom query в уже существующий механизм `postgres-exporter` (`charts/postgres-cluster/templates/configmap-exporter-queries.yaml`, тот же файл, что и `pg_extension` — см. [`docs/postgres-cluster-deployment-with-monitoring.md`](postgres-cluster-deployment-with-monitoring.md)), а не отдельный экспортёр:

```yaml
pg_partman_partitions:
  query: |
    SELECT
      current_database()                                             AS datname,
      pc.parent_table,
      (SELECT count(*) FROM partman.show_partitions(pc.parent_table)) AS partition_count
    FROM partman.part_config pc;
  metrics:
    - datname: {usage: "LABEL", description: "..."}
    - parent_table: {usage: "LABEL", description: "..."}
    - partition_count: {usage: "GAUGE", description: "..."}
```

Падение значения `pg_partman_partitions_partition_count` по конкретной `parent_table` = сработал детач; рост = сработал `premake` (`pg_partman` заранее создаёт будущие партиции) — обе панели показывают это на одном таймсерийном графике.

> **Важно (грабли, проверено на живом кластере):** `PG_EXPORTER_AUTO_DISCOVER_DATABASES=true` (уже включено в этом чарте) заставляет `postgres-exporter` выполнять **каждый** custom query против **каждой** обнаруженной базы, включая `postgres`/`template1`, где схемы `partman` нет вообще. Запрос, ссылающийся на `partman.part_config` напрямую в `FROM`, там неизбежно падает с `relation "partman.part_config" does not exist`. Проверено, что это **не фатально** — экспортёр логирует ошибку (`level=info ... Error running query on database ...`) и просто пропускает метрику для этой конкретной базы на этом цикле скрейпа, метрики для `app` (где схема есть) собираются нормально. Ничего оборачивать в try/except или ограничивать список баз не потребовалось.

После правки `queries.yaml` требуется рестарт подов кластера (реплики → лидер последним, как в [`postgres-walg-exporter-monitoring.md`](postgres-walg-exporter-monitoring.md)) — `postgres-exporter` читает файл конфигурации один раз при старте, «на лету» изменения не подхватывает:

```bash
kubectl delete pod postgres-cluster-1 postgres-cluster-2 -n postgres   # реплики (роли уточнить через /patroni)
kubectl wait --for=condition=Ready pod postgres-cluster-1 postgres-cluster-2 -n postgres --timeout=120s
kubectl delete pod postgres-cluster-0 -n postgres                      # текущий лидер — последним
kubectl wait --for=condition=Ready pod postgres-cluster-0 -n postgres --timeout=120s
```

### Метрики самого CronJob'а

`kube_cronjob_status_last_schedule_time` и `kube_job_status_succeeded`/`kube_job_status_failed` — из `kube-state-metrics`, который в этом стенде уже разворачивается как часть `victoria-metrics-k8s-stack` (ничего дополнительно ставить не нужно). Проверено на живом кластере: `kube_cronjob_status_last_schedule_time` заполняется **только** когда сам контроллер `CronJob` реально triggerит запуск по расписанию — ручной `kubectl create job --from=cronjob/...` (см. раздел 4) на это поле не влияет, только настоящее срабатывание `spec.schedule`.

## 7. Известные особенности

| Проблема | Решение |
|---|---|
| `psql -c "SELECT :'x';"` падает с `syntax error at or near ":"`, хотя `-v x=hello` явно установлена (`\echo :x` печатает `hello` без ошибок) | Подстановка `:'var'`/`:var`/`:"var"` в psql применяется только к запросу, который psql **читает из stdin** — при передаче через `-c` подстановка молча не срабатывает. Проверено эмпирически на живом кластере (PostgreSQL 18.2, тот же образ Spilo). Скрипт передаёт весь SQL через `stdin` (`subprocess.run(..., input=sql)`), никогда через `-c` |
| Владелец партиционированных таблиц (`app_owner`) — NOLOGIN, под ним нельзя подключиться | Job аутентифицируется как `postgres` (существующий секрет `postgres.<clusterName>.credentials.postgresql.acid.zalan.do`), не как владелец таблиц |
| TCP-подключение к Service кластера без `PGSSLMODE=require` отклоняется на этапе `pg_hba.conf` (`hostnossl all all all reject`) | Явно выставить `PGSSLMODE=require` в env контейнера — по умолчанию psql/libpq на `sslmode=prefer`, чего для этого `pg_hba.conf` недостаточно |
| Наивный парсинг суффикса `_pYYYYMMDD` регэкспом сработал бы только для суточного интервала и сломался бы на `p_interval => '1 week'`/`'1 month'` | Использовать `partman.show_partition_info()`, которая берёт границы партиции из реального констрейнта, а не из имени таблицы — работает для любого интервала |
| Голый `ALTER TABLE ... DETACH PARTITION`, написанный вручную, не учитывает защиту "не отсоединять последнюю оставшуюся партицию набора" | `partman.drop_partition_time()` уже содержит эту защиту (`RAISE WARNING ... CONTINUE`, если в наборе осталась ровно одна партиция) — переиспользуется готовая, а не своя реализация |
| Custom query `pg_partman_partitions` падает на базах без схемы `partman` (`PG_EXPORTER_AUTO_DISCOVER_DATABASES=true` гоняет её по всем базам) | Не фатально для остальных метрик/баз — `postgres-exporter` логирует ошибку и пропускает метрику только для этой базы на этом цикле (проверено на живом кластере) |
| Панель "Detached Partitions (last 24h)" показывает `0` сразу после добавления метрики, даже если детач только что произошёл | `delta()` считает разницу внутри окна `[24h]` — если метрика появилась в VictoriaMetrics позже, чем случился детач, более раннего (большего) значения в истории просто нет. Это не баг: после суток непрерывного сбора метрики панель считает корректно |
| Правка `configmap-exporter-queries.yaml` не подхватывается на лету | `postgres-exporter` читает `queries.yaml` один раз при старте контейнера — нужен рестарт пода. С [`docs/reloader-setup.md`](reloader-setup.md) это происходит автоматически (в течение ~5 минут, `repair_period` оператора) — без него нужен ручной рестарт (реплики → лидер последним) |

## 8. Дальнейшие шаги (не входят в эту инструкцию)

- Удаление отсоединённых, но уже подтверждённо заархивированных таблиц (`DROP TABLE`) — отдельная задача, требующая отдельного механизма учёта "какие таблицы уже безопасно удалять" (например, маркер после N дней с момента детача).
- Восстановление партиции из архива: `pg_restore` в новую таблицу + `ALTER TABLE parent ATTACH PARTITION` с явным диапазоном (`FOR VALUES FROM ... TO ...`) — вне рамок этой инструкции.
- Per-table retention (сейчас один глобальный `retention` на все таблицы, найденные в `partman.part_config`).
