# Полное развёртывание с нуля: PostgreSQL 18 + MinIO + мониторинг

Единая пошаговая инструкция, которая собирает воедино доки [2](postgres-cluster-deployment-with-monitoring.md), [3](postgres-walg-backup-setup.md), [6](postgres-pg-partman-setup.md) и [7](postgres-walg-exporter-monitoring.md) в один линейный сценарий — от пустого `kind`-кластера до полностью рабочего стенда: PostgreSQL 18 (3 инстанса), бэкапы через WAL-G в MinIO, мониторинг VictoriaMetrics + Grafana и таблицы `pg_partman` с данными.

В отличие от нумерованных доков (которые проходят исторический путь: сначала кластер на v14, потом апгрейд на v18, потом чистка конфига), здесь кластер сразу поднимается на v18 всеми нужными компонентами — так, как они уже лежат в манифестах репозитория. Здесь — только happy path и минимум объяснений «что и зачем»; за разбором конкретных граблей и альтернатив — ссылки на исходные доки.

ClickHouse в эту инструкцию не входит — он развёртывается независимо, см. [clickhouse-monitoring-stack-on-kind.md](clickhouse-monitoring-stack-on-kind.md).

## Что получим на выходе

- `kind`-кластер: 1 control-plane + 3 worker
- VictoriaMetrics + Grafana (сбор метрик и визуализация)
- MinIO — S3-совместимое хранилище для бэкапов
- PostgreSQL 18, 3 инстанса (1 Leader + 2 Replica через Patroni), под управлением Zalando `postgres-operator`
- Непрерывный WAL-архив и cron base backup через WAL-G в MinIO
- Полный мониторинг кластера: `prometheus-postgres-exporter`, `walg-exporter`, Patroni REST API, два дашборда в Grafana
- `pg_partman` с тремя таблицами (`events_daily`, `metrics_daily`, `request_logs_daily`), партиционированными по дню, с недельным объёмом тестовых данных

## Почему именно такой порядок шагов

Оператор PostgreSQL создаёт поды кластера **сразу же** при `kubectl apply` манифеста `postgresql`, и на старте пода все нужные ему объекты (секрет с кредами S3, конфигмап с общими параметрами WAL-G, CRD с поддержкой нужной версии) уже должны существовать — иначе получите либо под без WAL-архивирования (переменные не подхватились), либо ошибку валидации версии. Поэтому: сначала инфраструктура и креды, потом оператор, потом CRD-патч, и только в самом конце — сам кластер.

## Предварительные требования

- Docker
- [kind](https://kind.sigs.k8s.io/)
- kubectl
- Helm

Все команды выполняются из корня репозитория.

## Шаг 1. kind-кластер

```bash
kind create cluster --config cluster/kind-config.yaml
kubectl wait --for=condition=Ready nodes --all --timeout=120s
kubectl get nodes
```

Три worker-ноды нужны, чтобы 3 реплики PostgreSQL разъехались по разным нодам, как в проде.

## Шаг 2. VictoriaMetrics + Grafana

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo add postgres-operator-charts https://opensource.zalando.com/postgres-operator/charts/postgres-operator
helm repo add minio https://charts.min.io/
helm repo update

helm install vm vm/victoria-metrics-k8s-stack \
  --namespace monitoring \
  --create-namespace \
  --values monitoring/vm-values.yaml
```

Разворачивает VMSingle (хранилище метрик), VMAgent (сборщик), VMAlert (алертинг) и Grafana. `monitoring/vm-values.yaml` уже включает `sidecar.dashboards` — дашборды из ConfigMap с лейблом `grafana_dashboard=1` подхватываются автоматически, без ручных импортов через UI.

## Шаг 3. MinIO и бакет для бэкапов

```bash
helm install minio minio/minio \
  --namespace minio \
  --create-namespace \
  --set rootUser=minioadmin \
  --set rootPassword=minioadmin \
  --set mode=standalone \
  --set persistence.size=5Gi \
  --set resources.requests.memory=512Mi

POD=$(kubectl get pods -n minio -l app=minio -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n minio "$POD" -- mc alias set local http://localhost:9000 minioadmin minioadmin
kubectl exec -n minio "$POD" -- mc mb local/postgres-backups
```

`minioadmin`/`minioadmin` и plain-text креды — только для dev-стенда (см. `README.md`, раздел «Важно»).

## Шаг 4. Namespace'ы и креды для WAL-G

```bash
kubectl create namespace postgres
kubectl create namespace postgres-operator

kubectl create secret generic walg-config \
  --namespace postgres \
  --from-literal=AWS_ACCESS_KEY_ID=minioadmin \
  --from-literal=AWS_SECRET_ACCESS_KEY=minioadmin \
  --from-literal=AWS_ENDPOINT=http://minio.minio.svc.cluster.local:9000 \
  --from-literal=AWS_S3_FORCE_PATH_STYLE=true \
  --from-literal=AWS_REGION=us-east-1 \
  --from-literal=WALG_S3_PREFIX=s3://postgres-backups/spilo/postgres-cluster
```

- `postgres-pod-config` (ConfigMap, ns `postgres-operator`) — общие для всех кластеров параметры WAL-G (`USE_WALG_BACKUP`, `BACKUP_SCHEDULE` и т.д.), передаются оператором как env всем подам. Больше не создаётся отдельной командой — это часть чарта `postgres-cluster` (`templates/configmap-pod-config.yaml`), применится вместе с CR на Шаге 6.
- `walg-config` (Secret, ns `postgres`, **в namespace самого кластера**, не оператора) — креды доступа к MinIO. Именно как переменные окружения (`pod_environment_secret`), а не volume-mount — это критично, см. [postgres-walg-backup-setup.md, шаг 5](postgres-walg-backup-setup.md#5-включение-s3wal-g-в-конфиге-оператора): если секрет смонтировать как volume, `configure_spilo.py` не увидит кредов в `os.environ`, не включит `USE_WALE`, и `archive_command` молча останется `/bin/true`.

## Шаг 5. postgres-operator и CRD под версию 18

```bash
helm install postgres-operator postgres-operator-charts/postgres-operator \
  --namespace postgres-operator \
  --values postgres/operator/postgres-operator-values.yaml

kubectl apply -f https://raw.githubusercontent.com/zalando/postgres-operator/master/charts/postgres-operator/crds/postgresqls.yaml
kubectl get crd postgresqls.acid.zalan.do \
  -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.postgresql.properties.version.enum}'
# Ожидаем: ["14","15","16","17","18"]
```

`postgres/operator/postgres-operator-values.yaml` задаёт:
- `configGeneral.docker_image: ghcr.io/zalando/spilo-18:4.1-p1` — оператор будет использовать образ с бинарниками PG18 для всех кластеров;
- `configMajorVersionUpgrade` — актуально только при апгрейде существующего кластера (см. [postgres-major-version-upgrade-14-to-18.md](postgres-major-version-upgrade-14-to-18.md)), для свежего деплоя сразу на v18 эта секция ни на что не влияет, но не мешает;
- `configKubernetes.pod_environment_configmap`/`pod_environment_secret` — подключает конфигмап (создаётся вместе с CR на Шаге 6 как часть чарта `postgres-cluster`) и секрет (создан на Шаге 4) ко всем подам кластера.

CRD чарта 1.15.1 по умолчанию поддерживает версии только до 17 — версия 18 добавлена в master-ветке оператора, патчим вручную.

> **Важно (грабли):** оператор при **каждом** (пере)старте своего пода сам ресинхронизирует CRD под встроенную в бинарник схему (только до v17) — патч выше слетает не только после `helm upgrade`, но и после любого `kubectl rollout restart deployment postgres-operator` (в т.ч. форсированного ресинка в шаге 6 ниже). Если после такого рестарта получаете `Unsupported value: "18"` — просто переприменяйте команду `kubectl apply -f .../postgresqls.yaml` ещё раз. Подробности: [postgres-major-version-upgrade-14-to-18.md, шаг 2](postgres-major-version-upgrade-14-to-18.md#шаг-2-обновить-crd-для-поддержки-версии-18).

## Шаг 6. Кластер PostgreSQL

```bash
helm upgrade --install postgres-cluster ./charts/postgres-cluster \
  --namespace postgres \
  --values charts/postgres-cluster/values-production.yaml
kubectl get pods -n postgres -w
```

Одним релизом разворачивается сразу CR `postgresql`, ConfigMap `postgres-pod-config`, headless Service и VMServiceScrape. `values-production.yaml` — уже финальное состояние, включающее всё сразу (в отличие от историчеcкого пути доков 2→3→4→6→7, который к тому же финалу приходит через накопление `values-step*.yaml`, см. [postgres-cluster-deployment-with-monitoring.md](postgres-cluster-deployment-with-monitoring.md)):
- `numberOfInstances: 3`, `postgresql.version: "18"`;
- `preparedDatabases.app.extensions.pg_partman` — декларативная установка `pg_partman` в базу `app` (владелец `app_owner` в `databases:` специально совпадает с конвенцией `preparedDatabases`, иначе синхронизация упадёт правами — см. [postgres-pg-partman-setup.md](postgres-pg-partman-setup.md));
- `sidecars: prometheus-postgres-exporter, walg-exporter` — метрики Postgres и WAL-G;
- `postgresql.parameters.shared_preload_libraries` — включает `pg_cron` и `pg_partman_bgw`.

> **Важно (грабли): `pg_cron` в `shared_preload_libraries` обязателен именно при первом бутстрапе.** Встроенный `post_init.sh` образа Spilo безусловно пытается `CREATE EXTENSION pg_cron` в базе `postgres` при инициализации кластера — без `pg_cron` в preload это падает, и Patroni после 5 неудачных попыток останавливается насовсем (под виснет `3/3 Running`, но Patroni и Postgres внутри уже мертвы — под нужно удалить, штатный рестарт не поможет). В этом манифесте `pg_cron` уже оставлен в списке специально ради этого — не убирайте его без необходимости (подробности и как убрать безопасно на уже живом кластере — [postgres-pg-partman-setup.md, раздел про shared_preload_libraries](postgres-pg-partman-setup.md#оставляйте-в-shared_preload_libraries-только-то-что-реально-используется)).

**Первый прогон долгий** — образ Spilo (~560 МБ) качается независимо на каждую из 3 worker-нод, может занять 5-10+ минут в сумме. Дождитесь, пока все три пода станут `3/3 Running`:

```bash
kubectl get pods -n postgres
kubectl get postgresql -n postgres
```

Если `kubectl get postgresql` показывает `CreateFailed` при том, что все поды уже `3/3 Running` — это гонка: оператор ждёт готовности StatefulSet ограниченное число попыток и на медленном первом бутстрапе успевает истечь таймаутом раньше, чем поднимутся все поды. Форсируйте полный ресинк (и переприменит CRD-патч из шага 5 после него, см. предупреждение выше):

```bash
kubectl rollout restart deployment postgres-operator -n postgres-operator
kubectl rollout status deployment postgres-operator -n postgres-operator --timeout=60s
kubectl apply -f https://raw.githubusercontent.com/zalando/postgres-operator/master/charts/postgres-operator/crds/postgresqls.yaml
kubectl get postgresql -n postgres
# STATUS: Running
```

Проверьте версию и роли:

```bash
kubectl exec -it postgres-cluster-0 -n postgres -c postgres -- psql -U postgres -c "SELECT version();"
kubectl exec -it postgres-cluster-0 -n postgres -c postgres -- bash -c "patronictl -c /run/postgres.yml list"
```

## Шаг 7. Проверка WAL-G

```bash
kubectl exec -n postgres postgres-cluster-0 -c postgres -- printenv | grep -E "WALG_S3_PREFIX|AWS_ACCESS_KEY_ID"
kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -c "show archive_command;"
```

`archive_command` должен быть `envdir "/run/etc/wal-e.d/env" wal-g wal-push "%p"`, а не `/bin/true`. Тестовый base backup:

```bash
kubectl exec -n postgres postgres-cluster-0 -c postgres -- sh -c \
  'PGUSER=postgres PGHOST=/var/run/postgresql envdir /run/etc/wal-e.d/env wal-g backup-push $PGDATA'
```

Проверка бакета:

```bash
MINIO_POD=$(kubectl get pods -n minio -l app=minio -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n minio "$MINIO_POD" -- mc du local/postgres-backups
```

Полный разбор диагностики (нерабочий `archive_command`, протухшие таймлайны и т.д.) — [postgres-walg-backup-setup.md, раздел 7](postgres-walg-backup-setup.md#7-проверка).

## Шаг 8. Мониторинг Postgres

Уже развёрнут вместе с CR на Шаге 6 — чарт `postgres-cluster` одним релизом создаёт headless Service (порты `9187` pg-exporter, `8008` Patroni, `9351` walg-exporter), VMServiceScrape и ConfigMap с кастомным SQL-запросом к `pg_extension`. Проверить:

```bash
kubectl get vmservicescrape -n monitoring postgres-cluster
# STATUS: operational
```

## Шаг 9. Дашборды

```bash
helm upgrade --install monitoring-extras ./charts/monitoring-extras \
  --namespace monitoring \
  --values charts/monitoring-extras/values.yaml
```

Разворачивает ConfigMap для каждого файла из `charts/monitoring-extras/dashboards/*.json` (в т.ч. `postgresql-cluster-overview.json` и `postgresql-walg.json`), ClickHouse datasource(s) и VMServiceScrape для `clickhouse-operator` — последние два до развёртывания ClickHouse (см. [clickhouse-monitoring-stack-on-kind.md](clickhouse-monitoring-stack-on-kind.md)) просто не найдут целей, это ожидаемо.

Дашборды появятся в Grafana (папка `Databases`) через 15-20 секунд:

```bash
kubectl port-forward -n monitoring svc/vm-grafana 3000:80
# http://localhost:3000, admin/admin
```

## Шаг 10. pg_partman: таблицы и данные за неделю

Расширение `pg_partman` уже установлено в базу `app` декларативно манифестом кластера (шаг 6). Создаём партиционированные таблицы и наполняем данными:

```bash
kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app <<'EOF'
CREATE TABLE public.events_daily (
    id          bigserial,
    event_time  timestamptz NOT NULL DEFAULT now(),
    event_type  text NOT NULL,
    payload     jsonb,
    PRIMARY KEY (id, event_time)
) PARTITION BY RANGE (event_time);

SELECT partman.create_parent(
    p_parent_table     => 'public.events_daily',
    p_control          => 'event_time',
    p_interval         => '1 day',
    p_premake          => 7,
    p_start_partition  => (current_date - 6)::text
);

CREATE TABLE public.metrics_daily (
    id          bigserial,
    ts          timestamptz NOT NULL DEFAULT now(),
    metric_name text NOT NULL,
    value       double precision NOT NULL,
    PRIMARY KEY (id, ts)
) PARTITION BY RANGE (ts);

SELECT partman.create_parent(
    p_parent_table     => 'public.metrics_daily',
    p_control          => 'ts',
    p_interval         => '1 day',
    p_premake          => 7,
    p_start_partition  => (current_date - 6)::text
);

CREATE TABLE public.request_logs_daily (
    id          bigserial,
    ts          timestamptz NOT NULL DEFAULT now(),
    method      text NOT NULL,
    path        text NOT NULL,
    status_code integer NOT NULL,
    duration_ms integer NOT NULL,
    PRIMARY KEY (id, ts)
) PARTITION BY RANGE (ts);

SELECT partman.create_parent(
    p_parent_table     => 'public.request_logs_daily',
    p_control          => 'ts',
    p_interval         => '1 day',
    p_premake          => 7,
    p_start_partition  => (current_date - 6)::text
);

CALL partman.run_maintenance_proc();

INSERT INTO public.events_daily (event_time, event_type, payload)
SELECT
    d::date + (random() * interval '1 day'),
    (ARRAY['login','logout','click','purchase','error'])[floor(random()*5+1)],
    jsonb_build_object('user_id', floor(random()*10000)::int, 'session', md5(random()::text))
FROM generate_series(current_date - 6, current_date, interval '1 day') d,
     generate_series(1, 2000) g;

INSERT INTO public.metrics_daily (ts, metric_name, value)
SELECT
    d::date + (random() * interval '1 day'),
    (ARRAY['cpu_usage','memory_usage','disk_io','network_in','network_out'])[floor(random()*5+1)],
    round((random()*100)::numeric, 2)
FROM generate_series(current_date - 6, current_date, interval '1 day') d,
     generate_series(1, 2000) g;

INSERT INTO public.request_logs_daily (ts, method, path, status_code, duration_ms)
SELECT
    d::date + (random() * interval '1 day'),
    (ARRAY['GET','POST','PUT','DELETE'])[floor(random()*4+1)],
    (ARRAY['/api/users','/api/orders','/api/products','/health','/api/login'])[floor(random()*5+1)],
    (ARRAY[200,201,400,404,500])[floor(random()*5+1)],
    floor(random()*500+5)::int
FROM generate_series(current_date - 6, current_date, interval '1 day') d,
     generate_series(1, 2000) g;
EOF
```

`p_premake => 7` заранее создаёт партиции на неделю вперёд, `p_start_partition` — от `current_date - 6`, чтобы партиции нашлись и для вставляемых задним числом данных. `run_maintenance_proc()` — процедура (`CALL`, не `SELECT`). Подробный разбор граблей (алиасы интервалов, PK, схема расширения) — [postgres-pg-partman-setup.md](postgres-pg-partman-setup.md).

Автообслуживание партиций (`pg_partman_bgw`) уже включено в манифесте кластера — проверить воркер:

```bash
kubectl exec -n postgres postgres-cluster-0 -c postgres -- bash -c "ps aux | grep 'pg_partman master'"
```

## Финальная проверка всего стека

| Компонент | Команда | Ожидаемо |
|---|---|---|
| Ноды kind | `kubectl get nodes` | 4 × `Ready` |
| PostgreSQL кластер | `kubectl get postgresql -n postgres` | `VERSION 18`, `STATUS Running` |
| Поды кластера | `kubectl get pods -n postgres` | все `3/3 Running` |
| Роли Patroni | `patronictl -c /run/postgres.yml list` (внутри пода) | 1 Leader + 2 Replica, `streaming`, lag 0 |
| `archive_command` | `psql -c "show archive_command;"` | НЕ `/bin/true` |
| Бэкапы в MinIO | `mc du local/postgres-backups` | base backup + WAL-сегменты |
| `pg_partman` | `psql -d app -c "\dx"` | `pg_partman 5.4.1` |
| Данные в таблицах | `SELECT count(*) FROM public.events_daily;` | `14000` (и аналогично для двух других) |
| `pg_partman_bgw` | `ps aux \| grep 'pg_partman master'` | воркер запущен |
| VMServiceScrape | `kubectl get vmservicescrape -n monitoring postgres-cluster` | `operational` |
| Дашборды в Grafana | `curl -s -u admin:admin http://localhost:3000/api/search` | `PostgreSQL Cluster Overview`, `PostgreSQL WAL-G Backup` в папке `Databases` |

## Что дальше

- Апгрейд с v14 на v18 (если стартуете не сразу с v18) — [postgres-major-version-upgrade-14-to-18.md](postgres-major-version-upgrade-14-to-18.md)
- ClickHouse + мониторинг — [clickhouse-monitoring-stack-on-kind.md](clickhouse-monitoring-stack-on-kind.md)
- Глубокий разбор конкретных граблей по каждому шагу — соответствующие нумерованные доки, на которые здесь стоят ссылки
