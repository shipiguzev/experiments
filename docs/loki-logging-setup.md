# Логи: Loki + Promtail на kind

Инструкция по развёртыванию Grafana Loki (хранилище логов) и Promtail (агент сбора логов со всех подов кластера) поверх уже развёрнутого стека VictoriaMetrics + Grafana. Решает конкретную проблему: логи `CronJob`'ов (например, [архивирования партиций](postgres-partition-archive-setup.md)) живут ровно столько, сколько существует объект `Job`/`Pod` — после `successfulJobsHistoryLimit`/`failedJobsHistoryLimit` они пропадают безвозвратно. Loki хранит их отдельно от жизненного цикла самих подов, с полнотекстовым поиском (LogQL) через ту же Grafana.

## Область применения

Предполагается, что уже развёрнуты `kind`-кластер и [VictoriaMetrics + Grafana](victoriametrics-grafana-monitoring-on-kind.md) (namespace `monitoring`).

## Структура файлов

```
experiments/
├── monitoring/
│   ├── vm-values.yaml
│   ├── loki-values.yaml            # grafana/loki, SingleBinary + filesystem storage
│   └── promtail-values.yaml        # grafana/promtail, DaemonSet
└── charts/
    └── monitoring-extras/
        ├── values.yaml              # + loki.url
        └── templates/
            └── loki-datasource-cm.yaml
```

## 1. Почему SingleBinary + filesystem, а не object storage

Публичный чарт `grafana/loki` (v6.x) по умолчанию разворачивается в режиме `SimpleScalable` (`deploymentMode`), рассчитанном на объектное хранилище (S3/GCS) и раздельные read/write/backend компоненты — избыточно для локального стенда с парой ГБ логов в день. Используется `deploymentMode: SingleBinary` с `loki.storage.type: filesystem` — один под, локальный PVC, без S3/MinIO вообще (в отличие от WAL-G/clickhouse-backup/архивирования партиций в этом репозитории, которые сознательно используют MinIO как S3-совместимое хранилище).

Также отключены компоненты, ненужные для минимального деплоя:
- `gateway.enabled: false` — nginx-прокси между read/write путями нужен только в `SimpleScalable`/`Distributed`; в `SingleBinary` один под и так отдаёт push+query API на одном порту (3100).
- `minio.enabled: false` — у чарта есть свой встроенный MinIO для режимов с объектным хранилищем, не используется при `filesystem`.
- `chunksCache`/`resultsCache` (`enabled: false`) — по умолчанию это два memcached StatefulSet'а, один из которых (`chunksCache`) резервирует **8Gi** памяти — рассчитано на реальный трафик, для этого стенда явный оверкилл.
- `test.enabled: false`, `lokiCanary.enabled: false` — тестовые поды, не нужны для первого рабочего деплоя.

`charts/postgres-cluster` для WAL-G/архивирования партиций и `monitoring/vm-values.yaml` для VictoriaMetrics — оба ставятся напрямую из values-файлов репозитория; Loki/Promtail устанавливаются так же, отдельными релизами публичных чартов, без собственного `charts/`-обёртки (в отличие от `postgres-cluster`/`clickhouse-cluster`, где есть свой CR) — весь конфиг укладывается в values публичного чарта, добавлять шаблоны не потребовалось.

## 2. Установка Loki

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update grafana
```

```yaml
# monitoring/loki-values.yaml
deploymentMode: SingleBinary

loki:
  auth_enabled: false
  commonConfig:
    replication_factor: 1
  storage:
    type: filesystem
  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

singleBinary:
  replicas: 1
  persistence:
    size: 5Gi

read:
  replicas: 0
write:
  replicas: 0
backend:
  replicas: 0

gateway:
  enabled: false
minio:
  enabled: false
test:
  enabled: false
lokiCanary:
  enabled: false
chunksCache:
  enabled: false
resultsCache:
  enabled: false
```

```bash
helm upgrade --install loki grafana/loki --version 6.55.0 \
  --namespace monitoring \
  --values monitoring/loki-values.yaml

kubectl rollout status statefulset/loki -n monitoring --timeout=120s
```

Под `loki-0` должен стать `2/2 Running` (второй контейнер — `loki-sc-rules`, sidecar для подхвата `PrometheusRule`, не используется в этом стенде, но включён в чарте по умолчанию).

## 3. Установка Promtail

Promtail — DaemonSet, по одному поду на каждую ноду кластера, читает `/var/log/pods` через hostPath и пушит в Loki.

```yaml
# monitoring/promtail-values.yaml
config:
  clients:
    - url: http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push
```

> **Важно (грабли):** дефолтный `config.clients` в чарте `grafana/promtail` указывает на `http://loki-gateway/loki/api/v1/push` — сервис `loki-gateway` существует только при `deploymentMode: SimpleScalable`/`Distributed`. При `SingleBinary` (см. раздел 1) такого сервиса нет вообще — нужно явно перенаправить на сервис `loki` (порт `3100`, тот же под и так отдаёт push API).

```bash
helm upgrade --install promtail grafana/promtail --version 6.17.1 \
  --namespace monitoring \
  --values monitoring/promtail-values.yaml

kubectl rollout status daemonset/promtail -n monitoring --timeout=90s
```

Ожидается по одному поду `promtail` на каждую ноду (`kubectl get nodes` × `kubectl get pods -n monitoring -l app.kubernetes.io/name=promtail` — количество должно совпадать, в этом стенде 4: control-plane + 3 worker).

## 4. Datasource в Grafana

Тот же паттерн, что и для ClickHouse datasource ([`clickhouse-monitoring-stack-on-kind.md`](clickhouse-monitoring-stack-on-kind.md)) — ConfigMap с лейблом `grafana_datasource: "1"`, подхватывается sidecar'ом Grafana автоматически:

```yaml
# charts/monitoring-extras/templates/loki-datasource-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-datasource
  namespace: monitoring
  labels:
    grafana_datasource: "1"
data:
  loki-datasource.yaml: |
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki.monitoring.svc.cluster.local:3100
        jsonData:
          maxLines: 1000
```

```yaml
# charts/monitoring-extras/values.yaml
loki:
  url: http://loki.monitoring.svc.cluster.local:3100
```

```bash
helm upgrade monitoring-extras ./charts/monitoring-extras --namespace monitoring
```

## 5. Проверка

```bash
kubectl port-forward -n monitoring svc/vm-grafana 3000:80 &

# Datasource появился и здоров
DS_UID=$(curl -s -u admin:admin http://localhost:3000/api/datasources | python3 -c "import json,sys; print([d['uid'] for d in json.load(sys.stdin) if d['name']=='Loki'][0])")
curl -s -u admin:admin -X POST "http://localhost:3000/api/datasources/uid/$DS_UID/health"
# {"message":"Data source successfully connected.","status":"OK"}

kubectl port-forward -n monitoring svc/loki 3100:3100 &

# Есть логи хотя бы из одного namespace
curl -s --data-urlencode 'match[]={namespace=~".+"}' http://localhost:3100/loki/api/v1/series \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(sorted(set(s.get('namespace') for s in d['data'])))"

# Логи конкретно CronJob'а архивирования партиций
curl -s -G http://localhost:3100/loki/api/v1/query_range \
  --data-urlencode 'query={namespace="postgres", pod=~"postgres-cluster-partition-archive.*"}'
```

В Grafana UI логи доступны через **Explore** → датасорс `Loki`, запрос вида `{namespace="postgres", pod=~"postgres-cluster-partition-archive.*"}` (LogQL, тот же синтаксис фильтров по лейблам, что и в PromQL/MetricsQL, плюс `|= "text"` для полнотекстового фильтра по содержимому строки).

Реально проверено на этом стенде: логи `CronJob`'а архивирования партиций остаются доступными в Loki даже после того, как соответствующий объект `Job`/`Pod` уже удалён из кластера (в отличие от голого `kubectl logs`, см. [«Логи» в `postgres-partition-archive-setup.md`](postgres-partition-archive-setup.md#5-логи)).

## 6. Известные особенности

| Проблема | Решение |
|---|---|
| Дефолтный `config.clients` чарта `promtail` шлёт в `http://loki-gateway/...` | Сервис `loki-gateway` существует только в `SimpleScalable`/`Distributed` — при `SingleBinary` перенаправить на сервис `loki:3100` напрямую |
| Дефолтные `chunksCache`/`resultsCache` резервируют до 8Gi памяти под memcached | Отключить (`enabled: false`) для стенда такого масштаба — влияет только на латентность повторных запросов, не на работоспособность |
| `kubectl exec ... -- wget ...`/`curl` внутри пода `loki` — `executable file not found in $PATH` | В образе `grafana/loki` нет ни `wget`, ни `curl` — проверять API только через `kubectl port-forward` + `curl` с хоста |
| Grafana datasource proxy (`/api/ds/query`) без явного `from`/`to` в теле запроса возвращает пустой `frames: []` без ошибки | В отличие от прямого `/loki/api/v1/query_range` (у которого есть разумный дефолтный диапазон), `/api/ds/query` требует explicit временной диапазон в самом теле запроса |

## 7. Дальнейшие шаги (не входят в эту инструкцию)

- Retention логов (сейчас не настроен явно — используется дефолт чарта) и compaction для `filesystem` storage.
- Переключение хранилища на MinIO (`loki.storage.type: s3`, тот же инстанс, что уже используется для WAL-G/бэкапов ClickHouse/архивов партиций) — если объём логов вырастет за пределы удобного для локального PVC.
- Дашборд для самой Loki (`monitoring.dashboards.enabled: true` в чарте) — сейчас логи смотрятся только через Explore.
