# ClickHouse + мониторинг на kind

Инструкция по развёртыванию ClickHouse с мониторингом через VictoriaMetrics и Grafana в локальном Kubernetes кластере (kind).

## Предварительные требования

Перед началом убедитесь, что установлены:

- Docker
- [kind](https://kind.sigs.k8s.io/)
- kubectl
- Helm

Grafana Stack развёрнут согласно инструкции [VictoriaMetrics + Grafana Monitoring Stack на kind](victoriametrics-grafana-monitoring-on-kind.md).

## Структура файлов

```
experiments/
├── cluster/
│   └── kind-config.yaml
├── monitoring/
│   ├── vm-values.yaml
│   ├── clickhouse-datasource-cm.yaml
│   └── dashboards/
│       └── Altinity_ClickHouse_Operator_dashboard.json
└── clickhouse/
    ├── operator/
    │   └── clickhouse-operator-values.yaml
    └── installations/
        ├── chi-test.yaml
        └── monitoring/
            └── vmservicescrape-clickhouse-operator.yaml
```

## Шаг 1. Создание kind кластера

Создание кластера описано в инструкции [VictoriaMetrics + Grafana Monitoring Stack на kind](victoriametrics-grafana-monitoring-on-kind.md).

## Шаг 2. Установка clickhouse-operator

`clickhouse-operator` от Altinity управляет жизненным циклом ClickHouse кластеров в Kubernetes — создаёт поды, сервисы, конфигурации и следит за состоянием инсталляций через custom resource `ClickHouseInstallation` (CHI).

В values мы задаём два ключевых параметра:

- `watch.namespaces.include: [clickhouse]` — оператор будет следить только за namespace `clickhouse`. Без этого параметра оператор смотрит только на свой собственный namespace и не увидит CHI в других namespace.
- `metrics.enabled: true` — включает экспорт метрик оператора в Prometheus-формате. Метрики доступны на портах `ch-metrics` (8888) и `op-metrics` (9999) и содержат информацию о состоянии всех ClickHouse инсталляций.

```bash
mkdir -p clickhouse/operator
```

```yaml
# clickhouse/operator/clickhouse-operator-values.yaml
configs:
  files:
    config.yaml:
      watch:
        namespaces:
          include:
            - clickhouse
metrics:
  enabled: true
```

Добавьте репозиторий и установите оператор:

```bash
helm repo add clickhouse-operator https://docs.altinity.com/clickhouse-operator/
helm repo update
helm install clickhouse-operator clickhouse-operator/altinity-clickhouse-operator \
  --namespace clickhouse-operator \
  --create-namespace \
  --values clickhouse/operator/clickhouse-operator-values.yaml
```

Проверьте что оператор запущен:

```bash
kubectl get pods -n clickhouse-operator
```

Под должен быть `2/2 Running` — второй контейнер отвечает как раз за экспорт метрик (`metrics.enabled: true`).

## Шаг 3. Деплой ClickHouseInstallation

`ClickHouseInstallation` (CHI) — это custom resource, который описывает желаемое состояние ClickHouse кластера. Оператор читает этот манифест и создаёт все необходимые объекты: StatefulSet, Pod, Service, ConfigMap.

В манифесте определяются:

**Пользователь для мониторинга** — создаётся отдельный пользователь `monitoring` с паролем вместо использования пользователя `default`. Это важно по нескольким причинам:

- пользователь `default` по умолчанию ограничен сетевым доступом только с localhost и IP самих подов ClickHouse
- профиль `readonly` запрещает любые изменения данных — Grafana получает доступ только на чтение
- сетевая политика `::/0` разрешает подключение с любого IP внутри кластера, что необходимо для Grafana pod

**Топология кластера** — 1 шард и 2 реплики. Оператор создаст два пода: `chi-chi-test-test-0-0-0` и `chi-chi-test-test-0-1-0`. Для production репликация требует ZooKeeper или ClickHouse Keeper, в данном примере реплики независимы.

```bash
mkdir -p clickhouse/installations/monitoring
```

```yaml
# clickhouse/installations/chi-test.yaml
apiVersion: "clickhouse.altinity.com/v1"
kind: ClickHouseInstallation
metadata:
  name: chi-test
  namespace: clickhouse
spec:
  configuration:
    users:
      # Создаём отдельного пользователя для мониторинга
      monitoring/password: "monitoring"
      # Разрешаем подключение с любого IP внутри кластера
      monitoring/networks/ip:
        - "::/0"
      # Профиль readonly — только чтение, без изменения данных
      monitoring/profile: readonly
    clusters:
      - name: "test"
        layout:
          shardsCount: 1
          replicasCount: 2
```

Примените манифест:

```bash
kubectl create namespace clickhouse
kubectl apply -f clickhouse/installations/chi-test.yaml
```

Следите за статусом:

```bash
kubectl get chi -n clickhouse -w
```

Дождитесь статуса `Completed`. Первый прогон занимает пару минут — оператор последовательно поднимает оба пода (`chi-chi-test-test-0-0-0`, затем `chi-chi-test-test-0-1-0`), каждый со своим PVC.

## Шаг 4. Настройка мониторинга

### VMServiceScrape для метрик оператора

`VMServiceScrape` — это custom resource VictoriaMetrics оператора, аналог `ServiceMonitor` в Prometheus. Он указывает VMAgent, откуда собирать метрики.

Данный манифест настраивает сбор метрик с сервиса `clickhouse-operator` в namespace `clickhouse-operator`:

- порт `ch-metrics` (8888) — метрики самих ClickHouse инсталляций: количество подов, статус, ошибки
- порт `op-metrics` (9999) — метрики самого оператора: количество обработанных CHI, время reconciliation

Манифест размещается в папке `clickhouse/installations/monitoring/`, так как относится к мониторингу конкретной инсталляции, а не к общей инфраструктуре мониторинга.

```yaml
# clickhouse/installations/monitoring/vmservicescrape-clickhouse-operator.yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: clickhouse-operator
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - clickhouse-operator
  selector:
    matchLabels:
      app.kubernetes.io/name: altinity-clickhouse-operator
      app.kubernetes.io/instance: clickhouse-operator
  endpoints:
    - port: ch-metrics
    - port: op-metrics
```

### Datasource для Grafana

ConfigMap с лейблом `grafana_datasource: "1"` автоматически подхватывается sidecar-контейнером Grafana и добавляет datasource без ручной настройки через UI. Это позволяет управлять datasources как кодом (GitOps-подход) и не терять конфигурацию при рестарте Grafana.

Datasource настраивается на подключение к ClickHouse через балансировщик `clickhouse-chi-test`, который автоматически создаётся оператором и распределяет запросы между репликами. Используется пользователь `monitoring`, созданный в предыдущем шаге.

```yaml
# monitoring/clickhouse-datasource-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: clickhouse-datasource
  namespace: monitoring
  labels:
    # Лейбл для автоматического подхвата sidecar контейнером Grafana
    grafana_datasource: "1"
data:
  clickhouse-datasource.yaml: |
    apiVersion: 1
    datasources:
      - name: chi-test
        type: vertamedia-clickhouse-datasource
        # Балансировщик созданный оператором для всех реплик кластера
        url: http://clickhouse-chi-test.clickhouse.svc.cluster.local:8123
        access: proxy
        basicAuth: true
        basicAuthUser: monitoring
        secureJsonData:
          basicAuthPassword: monitoring
        jsonData:
          usePost: true
          defaultDatabase: default
```

Примените оба манифеста:

```bash
kubectl apply -f clickhouse/installations/monitoring/vmservicescrape-clickhouse-operator.yaml
kubectl apply -f monitoring/clickhouse-datasource-cm.yaml
```

Проверьте, что VMServiceScrape в статусе `operational`:

```bash
kubectl get vmservicescrape -n monitoring clickhouse-operator
```

Проверьте, что datasource реально подключается (запрос выполняется от имени Grafana к ClickHouse через `basicAuth`):

```bash
kubectl port-forward -n monitoring svc/vm-grafana 3000:80 &
DS_UID=$(curl -s -u admin:admin http://localhost:3000/api/datasources | python3 -c "import json,sys; print([d['uid'] for d in json.load(sys.stdin) if d['name']=='chi-test'][0])")
curl -s -u admin:admin -X POST http://localhost:3000/api/datasources/uid/$DS_UID/health
# {"message":"OK","status":"OK"}
```

## Шаг 5. Деплой дашборда

Дашборды деплоятся аналогично datasource — через ConfigMap с лейблом `grafana_dashboard: "1"`. Sidecar-контейнер Grafana отслеживает такие ConfigMap и автоматически загружает JSON файлы дашбордов.

Такой подход позволяет:

- хранить дашборды в Git вместе с остальными манифестами
- автоматически восстанавливать дашборды после рестарта Grafana
- деплоить несколько дашбордов одной командой

```bash
mkdir -p monitoring/dashboards
curl -s https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/grafana-dashboard/Altinity_ClickHouse_Operator_dashboard.json \
  -o monitoring/dashboards/Altinity_ClickHouse_Operator_dashboard.json
kubectl create configmap altinity-clickhouse-operator-dashboard \
  --from-file=monitoring/dashboards/Altinity_ClickHouse_Operator_dashboard.json \
  --namespace monitoring \
  --dry-run=client -o yaml | \
kubectl label --local -f - grafana_dashboard=1 --dry-run=client -o yaml | \
kubectl apply -f -
```

Дашборд появится в Grafana автоматически через 15-20 секунд.

## Шаг 6. Доступ к Grafana

```bash
kubectl port-forward -n monitoring svc/vm-grafana 3000:80
```

Откройте [http://localhost:3000](http://localhost:3000)

- Login: `admin`
- Password: `admin`

## Проверка

| Компонент | Команда |
|---|---|
| Поды оператора | `kubectl get pods -n clickhouse-operator` (ожидаем `2/2 Running`) |
| Поды ClickHouse | `kubectl get pods -n clickhouse` (`chi-chi-test-test-0-0-0`, `chi-chi-test-test-0-1-0`, оба `1/1 Running`) |
| Статус CHI | `kubectl get chi -n clickhouse` (`STATUS: Completed`) |
| Сервисы ClickHouse | `kubectl get svc -n clickhouse` (среди прочих — балансировщик `clickhouse-chi-test`) |
| VMServiceScrape | `kubectl get vmservicescrape -n monitoring clickhouse-operator` (`operational`) |
| Таргеты VMAgent | `kubectl port-forward -n monitoring svc/vmagent-vm-victoria-metrics-k8s-stack 8429:8429` → http://localhost:8429/targets — оба порта (`ch-metrics`, `op-metrics`) должны быть `up` |
| Datasource в Grafana | `curl -s -u admin:admin http://localhost:3000/api/datasources` — должен быть `chi-test` типа `vertamedia-clickhouse-datasource` |
| Дашборд в Grafana | `curl -s -u admin:admin "http://localhost:3000/api/search?query=ClickHouse"` — должен вернуть `Altinity ClickHouse Operator Dashboard` |
