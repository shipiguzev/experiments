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
│       ├── altinity-clickhouse-operator-dashboard.json
│       └── altinity-clickhouse-queries-dashboard.json
└── clickhouse/
    ├── operator/
    │   └── clickhouse-operator-values.yaml
    └── installations/
        ├── chi-test.yaml
        ├── chi-test-2.yaml                    # второй кластер в отдельном namespace, см. "Несколько кластеров"
        └── monitoring/
            └── vmservicescrape-clickhouse-operator.yaml
```

## Шаг 1. Создание kind кластера

Создание кластера описано в инструкции [VictoriaMetrics + Grafana Monitoring Stack на kind](victoriametrics-grafana-monitoring-on-kind.md).

## Шаг 2. Установка clickhouse-operator

`clickhouse-operator` от Altinity управляет жизненным циклом ClickHouse кластеров в Kubernetes — создаёт поды, сервисы, конфигурации и следит за состоянием инсталляций через custom resource `ClickHouseInstallation` (CHI).

В values мы задаём два ключевых параметра:

- `watch.namespaces.include: [clickhouse, clickhouse-2]` — список namespace, за которыми следит оператор. Без этого параметра оператор смотрит только на свой собственный namespace и не увидит CHI в других namespace. Список нужно расширять при добавлении новых кластеров в новые namespace — см. [«Несколько кластеров в разных namespace»](#несколько-кластеров-в-разных-namespace).
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

Дашборды деплоятся аналогично datasource — через ConfigMap с лейблом `grafana_dashboard: "1"`. Sidecar-контейнер Grafana отслеживает такие ConfigMap и автоматически загружает JSON файлы дашбордов. Аннотация `grafana_folder=Databases` кладёт дашборд в папку `Databases` (требует `sidecar.dashboards.folderAnnotation` и `provider.foldersFromFilesStructure: true` в `monitoring/vm-values.yaml`, уже включено там же, где и для дашбордов PostgreSQL/WAL-G).

Такой подход позволяет:

- хранить дашборды в Git вместе с остальными манифестами
- автоматически восстанавливать дашборды после рестарта Grafana
- деплоить несколько дашбордов одной командой

```bash
mkdir -p monitoring/dashboards
curl -s https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/grafana-dashboard/Altinity_ClickHouse_Operator_dashboard.json \
  -o monitoring/dashboards/altinity-clickhouse-operator-dashboard.json
kubectl create configmap altinity-clickhouse-operator-dashboard \
  --from-file=monitoring/dashboards/altinity-clickhouse-operator-dashboard.json \
  --namespace monitoring \
  --dry-run=client -o yaml | \
kubectl label --local -f - grafana_dashboard=1 --dry-run=client -o yaml | \
kubectl annotate --local -f - grafana_folder=Databases --dry-run=client -o yaml | \
kubectl apply -f -
```

Дашборд появится в Grafana автоматически через 15-20 секунд.

### ClickHouse Queries dashboard

Второй дашборд из того же upstream-репозитория — [`ClickHouse_Queries_dashboard.json`](https://github.com/Altinity/clickhouse-operator/blob/master/grafana-dashboard/ClickHouse_Queries_dashboard.json) (в этом репозитории сохранён под именем `altinity-clickhouse-queries-dashboard.json`). В отличие от operator-дашборда (метрики самого оператора из Prometheus/VictoriaMetrics), этот показывает данные из `system.query_log` самого ClickHouse — топ медленных запросов, потребление памяти, ошибки, request rate — то есть требует не Prometheus, а сам ClickHouse datasource.

Все панели используют переменную `$db` — датасорс-переменную с фильтром по типу `vertamedia-clickhouse-datasource`. Если датасорс этого типа один (`chi-test`), Grafana подставляет его автоматически; при нескольких (см. [«Несколько кластеров в разных namespace»](#несколько-кластеров-в-разных-namespace)) — выбирается вручную в UI.

Переменные `$exported_namespace`/`$chi` (K8S Namespace / K8S Clickhouse Installation) связаны с `$db`: запрос `$exported_namespace` фильтрует метрики оператора по `chi="${db:text}"`, а `$chi` в свою очередь фильтрует по `exported_namespace="$exported_namespace"` — получается цепочка `$db → $exported_namespace → $chi`. Работает это благодаря соглашению из этого репозитория: имя Grafana-датасорса всегда совпадает с именем CHI (`chi-test` датасорс ↔ `chi="chi-test"` лейбл, `chi-test-2` ↔ `chi="chi-test-2"`), поэтому лейбл `chi` в метриках оператора однозначно резолвится из значения `$db`.

**Важно:** нужно именно `${db:text}`, а не просто `$db`. У переменной `db` (тип `datasource`) `value` — это **UID** датасорса (`P3BD5D6D86D49CEBA`), а не его имя; `$db` интерполируется в `value`, поэтому фильтр `chi="$db"` сравнивает лейбл с UID и никогда не совпадает — `$exported_namespace`/`$chi` молча остаются пустыми («None» в выпадающем списке). Модификатор `:text` заставляет Grafana подставить отображаемое имя (`chi-test`/`chi-test-2`), которое и совпадает с лейблом `chi`. Проверялось через debug-логи плагина (`GF_LOG_FILTERS=tsdb.prometheus:debug`) — без него ошибка не видна ни в UI, ни в обычных логах, а рендер через `grafana-image-renderer` в полном `kiosk`-режиме её тоже не ловит (полный kiosk скрывает панель переменных, и Grafana просто не резолвит то, что не отрисовывается; нужен `kiosk=tv`, который прячет только навигацию).

**Известные особенности upstream-JSON (требуют правки перед деплоем):**

- Переменные `type`, `user`, `query_kind` (тип `query`, датасорс `$db`) хранят `query` как обычную строку — legacy-формат Grafana. Плагин `vertamedia-clickhouse-datasource` не реализует миграцию такого формата, из-за чего при загрузке дашборда всплывает баннер `Templating / Failed to upgrade legacy queries`. Фикс — привести `query` к объектному виду `{"query": "<тот же SQL>", "refId": "<name>-Variable-Query"}`, как уже сделано для `exported_namespace`/`chi` и во всех остальных дашбордах репозитория.
- Переменные `exported_namespace` и `chi` ссылаются на `${DS_PROMETHEUS}` — Prometheus datasource из оригинального экспорта, которого в кластере нет. Строковая (не объектная) ссылка на несуществующий датасорс — это и есть основная причина баннера `Failed to upgrade legacy queries`: Grafana не может смигрировать её в объектный `{type, uid}` формат. Хотя сами переменные нигде не используются в запросах панелей, их нужно починить, иначе дашборд не грузится целиком. Фикс — заменить `datasource` на реальный объект `{"type": "prometheus", "uid": "VictoriaMetrics"}` (наш VictoriaMetrics datasource агрегирует ровно те метрики оператора — `chi_clickhouse_metric_*` с лейблами `exported_namespace`/`chi` — на которые эти переменные и рассчитаны).
- Панель «Reqs/s type: $type; user: $user; query kind: $query_kind» (id `14`) использует макрос `$rate(...)`, который в плагине `vertamedia-clickhouse-datasource` всегда разворачивается через ClickHouse-функцию `runningDifference()`. Начиная с определённой версии ClickHouse эта функция задепрекейчена и по умолчанию заблокирована (`DEPRECATED_FUNCTION: ... set allow_deprecated_error_prone_window_functions to enable it`) — панель тихо показывает «No data» без явной ошибки (старая Angular-панель не всплывает баннер на 500 от датасорса). **Не включайте** `allow_deprecated_error_prone_window_functions` — ClickHouse не просто устарел, а прямо предупреждает, что функция даёт некорректные результаты в распределённых/многоблочных запросах (а тут `cluster('all-sharded', ...)` — ровно такой случай). Фикс — переписать запрос панели на тот же безопасный паттерн с оконной функцией `lag(...) OVER (ORDER BY t)`, который уже использует соседняя панель «Top N request's rate» (бакетирование по `$interval` вместо хардкода).

Все три правки уже внесены в `monitoring/dashboards/altinity-clickhouse-queries-dashboard.json` в этом репозитории — при повторном скачивании чистого JSON из upstream их нужно будет применить заново.

Диагностика такого рода ошибок (панель молча показывает «No data») требует реального рендера — прямые запросы к ClickHouse через `datasource/proxy` с руками подобранными параметрами могут случайно воспроизводить другой (рабочий) вариант запроса. Надёжный способ — включить debug-логи плагина datasource (`kubectl set env deployment/vm-grafana -n monitoring GF_LOG_FILTERS="plugin.vertamedia-clickhouse-datasource:debug"`) и посмотреть логи `grafana-image-renderer` (см. [настройку рендерера](grafana-image-renderer-setup.md)) — headless Chromium логирует ошибки консоли браузера с полным URL, включая точный сгенерированный SQL:

```bash
kubectl logs -n monitoring -l app=grafana-image-renderer --tail=200 | grep -i "500\|error"
```

```bash
curl -s https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/grafana-dashboard/ClickHouse_Queries_dashboard.json \
  -o monitoring/dashboards/altinity-clickhouse-queries-dashboard.json
kubectl create configmap clickhouse-queries-dashboard \
  --from-file=monitoring/dashboards/altinity-clickhouse-queries-dashboard.json \
  --namespace monitoring \
  --dry-run=client -o yaml | \
kubectl label --local -f - grafana_dashboard=1 --dry-run=client -o yaml | \
kubectl annotate --local -f - grafana_folder=Databases --dry-run=client -o yaml | \
kubectl apply -f -
```

Проверить, что датасорс реально отвечает на запросы (тот же путь, что использует панель дашборда — proxy к ClickHouse через Grafana):

```bash
DS_UID=$(curl -s -u admin:admin http://localhost:3000/api/datasources | python3 -c "import json,sys; print([d['uid'] for d in json.load(sys.stdin) if d['name']=='chi-test'][0])")
curl -s -u admin:admin -G "http://localhost:3000/api/datasources/proxy/uid/$DS_UID/" \
  --data-urlencode "query=SELECT count() FROM system.query_log FORMAT JSON"
```

## Несколько кластеров в разных namespace

Один оператор умеет обслуживать много `ClickHouseInstallation` в разных namespace — под это уже рассчитаны и оператор, и оба дашборда. В репозитории вторым примером развёрнут `chi-test-2` в namespace `clickhouse-2` (`clickhouse/installations/chi-test-2.yaml`, кластер `test2`), чтобы явно проверить эту схему на практике.

Что нужно на каждый новый кластер/namespace:

1. **Namespace попадает в список оператора.** Добавить его в `watch.namespaces.include` в `clickhouse/operator/clickhouse-operator-values.yaml` и накатить `helm upgrade` — без этого оператор не увидит CHI в новом namespace.
2. **Новый CHI-манифест** — копия `chi-test.yaml` с другим `metadata.name`/`namespace` и, чтобы не путать метрики/логи, другим именем кластера (`spec.configuration.clusters[].name`).
3. **Отдельный Grafana datasource** — второй элемент в списке `datasources:` в `monitoring/clickhouse-datasource-cm.yaml`, с уникальным `name` и `url`, указывающим на балансировщик нового кластера (`clickhouse-<chi-name>.<namespace>.svc.cluster.local`).

Дальше ничего вручную донастраивать не нужно — оба дашборда уже параметризованы под multi-cluster:

- **Altinity ClickHouse Operator Dashboard** — единственный сервис оператора отдаёт метрики (`chi_clickhouse_metric_*`) сразу по всем CHI, которые он видит, с лейблами `exported_namespace`/`chi`. Переменные `$exported_namespace`/`$chi` на дашборде подхватывают новые значения автоматически — новый кластер просто появляется в выпадающих списках.
- **Altinity ClickHouse Queries Dashboard** — переменная `$db` (тип `datasource`, фильтр по плагину `vertamedia-clickhouse-datasource`) выводит все датасорсы этого типа, так что новый датасорс из шага 3 сразу становится доступен для выбора в дашборде без правки самого JSON.

Проверка, что новый кластер реально виден по обоим путям:

```bash
# метрики оператора по новому namespace/chi (после port-forward vmsingle на 8428)
curl -s "http://localhost:8428/api/v1/query?query=chi_clickhouse_metric_Uptime" | \
  python3 -c "import json,sys; [print(r['metric'].get('exported_namespace'), r['metric'].get('chi')) for r in json.load(sys.stdin)['data']['result']]"

# новый datasource и его health (после port-forward vm-grafana на 3000)
curl -s -u admin:admin http://localhost:3000/api/datasources | python3 -c "import json,sys; [print(d['name'], d['url']) for d in json.load(sys.stdin) if 'chi-test' in d['name']]"
```

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
| Дашборды в Grafana | `curl -s -u admin:admin "http://localhost:3000/api/search?query=ClickHouse"` — должны вернуться `Altinity ClickHouse Operator Dashboard` и `Altinity ClickHouse Queries Dashboard`, оба в папке `Databases` |
| Второй кластер (`chi-test-2`) | `kubectl get chi -n clickhouse-2` (`STATUS: Completed`), `kubectl get pods -n clickhouse-2` (оба пода `1/1 Running`) |
| Метрики второго кластера | `curl -s "http://localhost:8428/api/v1/label/chi/values"` (после port-forward vmsingle) — должны быть и `chi-test`, и `chi-test-2` |
