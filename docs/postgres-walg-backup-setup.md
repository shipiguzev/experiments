# Настройка бэкапа PostgreSQL через WAL-G в MinIO/S3

## Область применения

Инструкция описывает настройку резервного копирования (base backup + непрерывный WAL-архив) для кластера PostgreSQL, развёрнутого через [Zalando postgres-operator](https://github.com/zalando/postgres-operator), на S3-совместимое хранилище (MinIO) с помощью встроенного в образ Spilo инструмента **WAL-G**.

Предполагается, что на кластере уже развёрнуты:
- `postgres-operator` (helm-релиз `postgres-operator` в namespace `postgres-operator`);
- целевой Postgres-кластер (CR `postgresql.acid.zalan.do`), например `postgres-cluster` в namespace `postgres`;
- S3-совместимое хранилище (в нашем случае MinIO, namespace `minio`).

## 1. Развёртывание MinIO (если ещё не развёрнут)

Если S3-совместимое хранилище в кластере ещё отсутствует, поднимаем MinIO в standalone-режиме через helm.

```bash
helm repo add minio https://charts.min.io/
helm repo update

helm install minio minio/minio \
  --namespace minio \
  --create-namespace \
  --set rootUser=minioadmin \
  --set rootPassword=minioadmin \
  --set mode=standalone \
  --set persistence.size=5Gi \
  --set resources.requests.memory=512Mi

kubectl get pods -n minio
```

Проверяем, что под поднялся:

```bash
kubectl get pods -n minio
# NAME                    READY   STATUS    RESTARTS   AGE
# minio-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

Сервис доступен изнутри кластера по адресу `minio.minio.svc.cluster.local:9000`.

> Значения `rootUser`/`rootPassword` и размер `persistence.size` — для тестового/dev-окружения. Для прода использовать секреты вместо plain-text значений и рассмотреть `mode=distributed` для отказоустойчивости.

## 2. Подготовка бакета в MinIO

Создаём бакет, куда будут складываться бэкапы и WAL-сегменты.

```bash
POD=$(kubectl get pods -n minio -l app=minio -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n minio "$POD" -- mc alias set local http://localhost:9000 minioadmin minioadmin
kubectl exec -n minio "$POD" -- mc mb local/postgres-backups
kubectl exec -n minio "$POD" -- mc ls local
```

> Для реального S3 (не MinIO) этот шаг заменяется созданием бакета в соответствующем провайдере и IAM-пользователя/ключа с правами на запись в этот бакет.

## 3. Секрет с доступом к S3

Секрет создаётся **в namespace самого Postgres-кластера** (не в `postgres-operator`). Ключи секрета будут переданы во все поды кластера как обычные переменные окружения (через `pod_environment_secret`, см. шаг 5) — именно в таком виде их ожидает bootstrap-скрипт Spilo (`configure_spilo.py`), который по ним самостоятельно решает, включать ли WAL-архивирование, и сам генерирует файлы для `envdir`, которые затем читает `wal-g`.

```bash
kubectl create secret generic walg-config \
  --namespace postgres \
  --from-literal=AWS_ACCESS_KEY_ID=minioadmin \
  --from-literal=AWS_SECRET_ACCESS_KEY=minioadmin \
  --from-literal=AWS_ENDPOINT=http://minio.minio.svc.cluster.local:9000 \
  --from-literal=AWS_S3_FORCE_PATH_STYLE=true \
  --from-literal=AWS_REGION=us-east-1 \
  --from-literal=WALG_S3_PREFIX=s3://postgres-backups/spilo/postgres-cluster
```

> `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` здесь равны `rootUser`/`rootPassword` MinIO из шага 1 (`minioadmin`/`minioadmin`) — для dev-стенда отдельного IAM-пользователя не заводили. Для реального S3 это будут отдельные access/secret key IAM-пользователя с правами на бакет.

Пояснения по ключам:
| Ключ | Назначение |
|---|---|
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` | креды доступа к S3/MinIO |
| `AWS_ENDPOINT` | адрес MinIO внутри кластера, формат `http://<svc>.<namespace>.svc.cluster.local:9000` (в нашем случае `http://minio.minio.svc.cluster.local:9000`) |
| `AWS_S3_FORCE_PATH_STYLE` | обязательно `true` для MinIO (path-style, а не virtual-hosted) |
| `AWS_REGION` | для MinIO значение произвольное, но должно быть задано (например `us-east-1`) |
| `WALG_S3_PREFIX` | префикс в бакете, куда пишутся бэкапы конкретного кластера |

Для реального AWS S3 достаточно `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` и `WALG_S3_PREFIX` (без `AWS_ENDPOINT`/`AWS_S3_FORCE_PATH_STYLE`).

## 4. ConfigMap с общими параметрами WAL-G

Конфигмап с параметрами, одинаковыми для всех кластеров под управлением оператора. Живёт в namespace `postgres-operator` (путь указывается в конфиге оператора, см. шаг 5). Рендерится чартом `postgres-cluster` (`templates/configmap-pod-config.yaml`, namespace берётся из `values.operatorNamespace`) — значения ниже уже собраны в `charts/postgres-cluster/values-step3-walg.yaml`:

```yaml
podConfig:
  USE_WALG_BACKUP: "true"
  USE_WALG_RESTORE: "true"
  CLONE_USE_WALG_RESTORE: "true"
  WALG_DISABLE_S3_SSE: "true"      # для MinIO без SSE; для AWS S3 с шифрованием — убрать/false
  BACKUP_SCHEDULE: "0 2 * * *"     # cron автоматического base backup
  BACKUP_NUM_TO_RETAIN: "7"        # сколько последних бэкапов хранить
```

Накатите этот values-файл поверх уже установленного релиза (сам CR `postgresql` этот шаг не меняет — WAL-G на этом этапе включается только конфигом, см. шаг 5):

```bash
helm upgrade postgres-cluster ./charts/postgres-cluster \
  --namespace postgres \
  --values charts/postgres-cluster/values.yaml \
  --values charts/postgres-cluster/values-step3-walg.yaml
```

> **Важно:** `helm upgrade` не запоминает `-f`, переданные в предыдущих вызовах — на каждом следующем шаге этой цепочки доков нужно перечислять все `values-step*.yaml` с начала, а не только новый файл (иначе непереданные значения из более ранних шагов откатятся на дефолты чарта). Дальше по докам список файлов будет только расти.

## 5. Включение S3/WAL-G в конфиге оператора

В values-файле helm-релиза `postgres-operator` включаем секцию `configKubernetes`:

```yaml
# postgres-operator-values.yaml
configKubernetes:
  pod_environment_configmap: "postgres-operator/postgres-pod-config"
  pod_environment_secret: "walg-config"
```

- `pod_environment_configmap` — конфигмап (шаг 4) с общими env-переменными (namespace/name через `/`), живёт в namespace `postgres-operator`.
- `pod_environment_secret` — имя секрета (шаг 3) **в namespace самого кластера**; оператор инжектит все его ключи как переменные окружения (`envFrom`/`secretKeyRef`) во все Postgres-поды, которыми управляет.

> **Важно (грабли): не используйте `configAwsOrGcp.additional_secret_mount`/`additional_secret_mount_path` для этой цели, и не задавайте `wal_s3_bucket`.**
>
> Может показаться логичным просто примонтировать секрет как volume (`additional_secret_mount`) по пути `/run/etc/wal-e.d/env`, который читает `envdir` при вызове `wal-g`. Это действительно даёт `wal-g` доступ к кредам **при ручном вызове** (см. шаг 7.4), но **не включает автоматическое WAL-архивирование**: `archive_command` вычисляется `configure_spilo.py` при бутстрапе пода **только по переменным окружения процесса** (`os.environ` — `WAL_S3_BUCKET`/`WALE_S3_PREFIX`/`WALG_S3_PREFIX` и т.п.), а не по файлам внутри смонтированного секрета. Если ни одна из этих переменных не видна в окружении, `USE_WALE` остаётся `false`, и `configure_spilo.py` жёстко прописывает `archive_command = /bin/true` — WAL просто не архивируется, при этом под стартует без единой ошибки, а ручной `wal-g backup-push` из шага 7.4 продолжает работать, маскируя проблему.
>
> При этом добавление `wal_s3_bucket` (чтобы прокинуть `WAL_S3_BUCKET` в окружение и включить `USE_WALE`) при одновременно смонтированном `additional_secret_mount` тоже не работает и падает ещё раньше: `configure_spilo.py` (функция `write_wale_environment`) при виде `WAL_S3_BUCKET`/`WALG_S3_PREFIX`/... в окружении сам пытается дозаписать файлы (`WALE_S3_PREFIX` и др.) в директорию `WALE_ENV_DIR` — ту же `/run/etc/wal-e.d/env`, которая уже занята смонтированным секретом. Secret-volume в Kubernetes монтируется **read-only**, поэтому запись падает с `OSError: [Errno 30] Read-only file system: '/run/etc/wal-e.d/env/WALE_S3_PREFIX'`, `configure_spilo.py` крашится, и Patroni в поде вообще не стартует (см. диагностику в шаге 7).
>
> Правильное решение — `pod_environment_secret` (env, не volume): переменные видны `configure_spilo.py` при бутстрапе, `USE_WALE` включается, и скрипт **сам** создаёт `/run/etc/wal-e.d/env` со всеми нужными файлами (директория ещё не занята никаким volume) — `additional_secret_mount` в этой схеме не нужен вообще.

Применяем апгрейдом релиза:

```bash
helm upgrade postgres-operator postgres-operator-charts/postgres-operator \
  --namespace postgres-operator \
  --values postgres-operator-values.yaml
```

## 6. Форсированный полный ресинк оператора

**Важно (грабли):** после `helm upgrade` оператор иногда успевает выполнить синхронизацию кластера по гонке состояний — в логах видно, что диф со всеми новыми переменными/volume для WAL-G посчитан и запланирован (`reason: new statefulset containers's postgres environment does not match...`), но реально применённый StatefulSet эти изменения не содержит (ни volume `walg-config`, ни env `WAL_S3_BUCKET`/`BACKUP_SCHEDULE` и т.п.). Пересинхронизация происходит некорректно один раз сразу после апгрейда конфигурации оператора.

Лечится принудительным полным перезапуском оператора, который выполняет чистую полную ресинхронизацию всех кластеров при старте:

```bash
kubectl rollout restart deployment postgres-operator -n postgres-operator
kubectl rollout status deployment postgres-operator -n postgres-operator --timeout=60s
```

После этого оператор выполнит rolling update пода(ов) кластера — они по очереди пересоздадутся с новым StatefulSet.

## 7. Проверка

### 7.1. StatefulSet содержит нужные env

```bash
kubectl get statefulset postgres-cluster -n postgres -o json | \
  python3 -c "import json,sys; d=json.load(sys.stdin); c=[c for c in d['spec']['template']['spec']['containers'] if c['name']=='postgres'][0]; print([e['name'] for e in c.get('env',[]) if e.get('valueFrom',{}).get('secretKeyRef',{}).get('name')=='walg-config'])"
```

Ожидаем увидеть среди env-переменных с `secretKeyRef.name == walg-config`: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_ENDPOINT`, `WALG_S3_PREFIX` и т.д.

### 7.2. Все поды кластера перекатились и готовы

```bash
kubectl get pods -n postgres -l cluster-name=postgres-cluster
```

Все — `2/2 Running`.

> **Осторожно:** у контейнера `postgres` в образе Spilo нет readiness/liveness проб, поэтому `2/2 Running` означает только «процессы контейнера живы», а не «Patroni и Postgres реально запущены». Если конфигурация сломана (см. 7.6), под может висеть в `Running` сколь угодно долго с мёртвым Patroni внутри — полагаться нужно на проверки 7.3–7.6, а не на статус пода.

### 7.3. Переменные окружения реально видны процессу, и archive_command включён (на мастере)

Мастер после рестарта/failover может смениться, поэтому сначала определяем его динамически:

```bash
for p in postgres-cluster-0 postgres-cluster-1 postgres-cluster-2; do
  role=$(kubectl exec -n postgres "$p" -c postgres -- curl -s http://localhost:8008/patroni | python3 -c "import json,sys; print(json.load(sys.stdin).get('role'))")
  echo "$p: $role"
done
# пример вывода: postgres-cluster-2: primary
```

Далее используем найденный под (в нашем случае — `postgres-cluster-2`):

```bash
kubectl exec -n postgres postgres-cluster-2 -c postgres -- printenv | grep -E "WALG_S3_PREFIX|AWS_ACCESS_KEY_ID"
kubectl exec -n postgres postgres-cluster-2 -c postgres -- psql -U postgres -c "show archive_command;"
```

`archive_command` должен быть вида `envdir "/run/etc/wal-e.d/env" wal-g wal-push "%p"`, а **не** `/bin/true`. Если он `/bin/true` несмотря на то, что секрет с кредами создан и `pod_environment_secret` настроен — см. критичное предупреждение в шаге 5: скорее всего креды доступны только как файлы (через `additional_secret_mount`), а не как переменные окружения, которые нужны `configure_spilo.py` для включения `USE_WALE`.

Дополнительно можно проверить, что Spilo сам сгенерировал envdir-файлы (обычные файлы, а не симлинки — в отличие от подхода с `additional_secret_mount`):

```bash
kubectl exec -n postgres postgres-cluster-2 -c postgres -- ls -la /run/etc/wal-e.d/env/
```

### 7.4. Ручной тестовый backup-push

```bash
kubectl exec -n postgres postgres-cluster-2 -c postgres -- sh -c \
  'PGUSER=postgres PGHOST=/var/run/postgresql envdir /run/etc/wal-e.d/env wal-g backup-push $PGDATA'
```

> `PGUSER`/`PGHOST` нужно задать явно — при обычном `kubectl exec` они не наследуются из окружения самого процесса Patroni, и wal-g иначе пытается подключиться под системным пользователем шелла (в контейнере это `root`), получая `password authentication failed`.

Успешный вывод заканчивается строкой вида:
```
INFO: ... Wrote backup with name base_XXXXXXXXXXXXXXXXXXXXXXXX to storage default
```

### 7.5. Проверка содержимого бакета

```bash
MINIO_POD=$(kubectl get pods -n minio -l app=minio -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n minio "$MINIO_POD" -- mc ls --recursive local/postgres-backups
kubectl exec -n minio "$MINIO_POD" -- mc du local/postgres-backups
```

Ожидаем увидеть:
- `basebackups_005/base_.../tar_partitions/part_*.tar.lz4` — свежий base backup;
- `wal_005/*.lz4` — непрерывно накапливающиеся WAL-сегменты (архивируются автоматически через `archive_command`, без ручных действий).

### 7.6. `patronictl list` падает с "No cluster names were provided"

```
kubectl exec -it postgres-cluster-0 -n postgres -- bash -c "patronictl list"
2026-... - WARNING - Listing members: No cluster names were provided
```

Это симптом, а не причина: `patronictl` не находит scope, потому что конфиг Patroni (`/run/postgres.yml`) вообще не был сгенерирован — сам процесс Patroni не стартовал. Проверить:

```bash
kubectl logs -n postgres postgres-cluster-0 -c postgres --tail=50
```

Если в логах видно, что бутстрап падает на шаге `Configuring wal-e` с трейсбеком вида

```
OSError: [Errno 30] Read-only file system: '/run/etc/wal-e.d/env/WALE_S3_PREFIX'
```

— значит в конфиге оператора задан `wal_s3_bucket` вместе с `additional_secret_mount_path`, указывающим на ту же директорию (см. предупреждение в шаге 5). Убрать `wal_s3_bucket` из `postgres-operator-values.yaml`, повторить `helm upgrade` (шаг 5).

Дальше есть ещё одна ловушка: если Patroni уже был мёртв в поде на момент `helm upgrade`/рестарта оператора (как в этом сценарии), штатный rolling update оператора **не сработает** — он координирует пересоздание подов через Patroni REST API (`GET :8008/patroni`), а раз Patroni не отвечает, в логах оператора будет:

```
errors while restarting Postgres in pods via Patroni API: ... connect: connection refused
postpone pod recreation until next sync
```

и оператор будет откладывать пересоздание бесконечно. В этом случае StatefulSet уже обновлён правильно (проверить шагом 7.1) — поды нужно пересоздать вручную. Реплики — сначала, лидера — последним (Patroni сам проведёт failover на одну из уже пересозданных реплик):

```bash
kubectl delete pod postgres-cluster-1 postgres-cluster-2 -n postgres   # реплики
kubectl wait --for=condition=Ready pod postgres-cluster-1 postgres-cluster-2 -n postgres --timeout=120s
kubectl delete pod postgres-cluster-0 -n postgres                      # текущий лидер — последним
kubectl wait --for=condition=Ready pod postgres-cluster-0 -n postgres --timeout=120s
```

Так как PGDATA лежит на PVC, удаление подов безопасно — StatefulSet пересоздаст их по актуальному шаблону, Patroni поднимется заново на существующих данных. После пересоздания лидера — заново определите текущего мастера командой из шага 7.3, он мог смениться.

## 8. Автоматизация

После настройки:
- **Base backup** выполняется автоматически по cron внутри пода согласно `BACKUP_SCHEDULE` (`postgres-pod-config`), хранится последние `BACKUP_NUM_TO_RETAIN` штук.
- **WAL-архивирование** работает непрерывно и автоматически (не требует крона) сразу после того, как под получил переменные окружения из секрета `walg-config` (через `pod_environment_secret`) — обязательно проверить это шагом 7.3, а не полагаться на факт наличия секрета/конфигов.

Дополнительно ничего разворачивать не нужно — оба механизма встроены в образ Spilo.

## 9. Восстановление (кратко, для справки)

Восстановление из бэкапа делается либо через `wal-g backup-fetch`, либо декларативно через `clone`-секцию в манифесте `postgresql.acid.zalan.do` (создание нового кластера из бэкапа существующего). Требует отдельной инструкции — не входит в область данного документа.
