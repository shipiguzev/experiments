# Скриншоты дашбордов через grafana-image-renderer

Инструкция по развёртыванию `grafana-image-renderer` — сервиса для рендеринга дашбордов Grafana в PNG через `/render/...` API. Полезно для отладки: визуально проверить дашборд без реального браузера у оператора (например, автоматизированно после деплоя нового дашборда).

Предполагается, что стек VictoriaMetrics + Grafana уже развёрнут — см. [VictoriaMetrics + Grafana Monitoring Stack на kind](victoriametrics-grafana-monitoring-on-kind.md).

## Структура файлов

```
experiments/
├── monitoring/
│   └── vm-values.yaml                  # + GF_RENDERING_SERVER_URL/GF_RENDERING_CALLBACK_URL
└── charts/
    └── monitoring-extras/
        └── templates/grafana-image-renderer.yaml   # imageRenderer.enabled: false по умолчанию
```

## Почему отдельным сервисом, а не плагином

Штатный способ подключить рендеринг в Grafana — установить плагин `grafana-image-renderer` через `GF_INSTALL_PLUGINS`, как это сделано для `vertamedia-clickhouse-datasource`. Но этот плагин запускает headless Chromium в том же контейнере, что и сама Grafana, а образ `grafana/grafana:11.1.0`, используемый в этом стенде, собран на Alpine — headless Chromium на Alpine/musl не запускается.

Поэтому рендерер разворачивается отдельным сервисом на собственном образе (`grafana/grafana-image-renderer:3.12.4`, glibc + Chromium), а Grafana обращается к нему по HTTP через `GF_RENDERING_SERVER_URL`.

## Шаг 1. Деплой сервиса рендерера

Deployment+Service уже есть в чарте `monitoring-extras` (`templates/grafana-image-renderer.yaml`), но по умолчанию выключены (`imageRenderer.enabled: false`) — это единственный опциональный кусок чарта, включается явным флагом:

```bash
helm upgrade --install monitoring-extras ./charts/monitoring-extras \
  --namespace monitoring \
  --values charts/monitoring-extras/values.yaml \
  --set imageRenderer.enabled=true
kubectl rollout status deployment/grafana-image-renderer -n monitoring --timeout=180s
```

Образ рендерера довольно большой (бандлит Chromium) — первый pull может занять пару минут.

## Шаг 2. Подключение Grafana к рендереру

В `monitoring/vm-values.yaml` добавьте переменные окружения Grafana:

```yaml
# monitoring/vm-values.yaml
grafana:
  env:
    GF_RENDERING_SERVER_URL: "http://grafana-image-renderer.monitoring.svc.cluster.local:8081/render"
    GF_RENDERING_CALLBACK_URL: "http://vm-grafana.monitoring.svc.cluster.local:80/"
```

Примените изменения:

```bash
helm upgrade vm vm/victoria-metrics-k8s-stack \
  --namespace monitoring \
  --values monitoring/vm-values.yaml
kubectl rollout status deployment/vm-grafana -n monitoring --timeout=120s
```

## Шаг 3. Скриншот дашборда

```bash
kubectl port-forward -n monitoring svc/vm-grafana 3000:80 &
curl -s -o dashboard.png -u admin:admin \
  "http://localhost:3000/render/d/<uid>/<slug>?width=1400&height=1000&from=now-24h&to=now&kiosk"
file dashboard.png   # должен быть PNG, а не текст "No image renderer available/installed"
```

`<uid>`/`<slug>` дашборда можно взять из `curl -s -u admin:admin "http://localhost:3000/api/search?query=<название>"`.

## Отладка панелей через логи рендерера

Если панель на дашборде показывает «No data» без видимой ошибки (типично для старых Angular-панелей, которые глотают HTTP-ошибку датасорса), самих логов Grafana часто недостаточно — access-лог не пишет тело запроса/query. Рендерер логирует ошибки консоли браузера headless Chromium, включая полный URL запроса (а значит и точный SQL/PromQL, который реально ушёл в датасорс):

```bash
kubectl logs -n monitoring -l app=grafana-image-renderer --tail=200 | grep -i "500\|error"
```

Это надёжнее, чем руками подбирать параметры запроса через `datasource/proxy` — велик шанс случайно воспроизвести другой (рабочий) вариант запроса вместо реально исполняемого дашбордом.

Для более подробных логов самого datasource-плагина:

```bash
kubectl set env deployment/vm-grafana -n monitoring GF_LOG_FILTERS="plugin.<plugin-id>:debug"
kubectl rollout status deployment/vm-grafana -n monitoring --timeout=90s
# ... воспроизвести рендер ...
kubectl set env deployment/vm-grafana -n monitoring GF_LOG_FILTERS-   # снять debug-уровень
```

## Проверка

| Компонент | Команда |
|---|---|
| Под рендерера | `kubectl get pods -n monitoring -l app=grafana-image-renderer` (`1/1 Running`) |
| Рендеринг работает | `curl -s -o test.png -u admin:admin "http://localhost:3000/render/d/<uid>/<slug>?width=800&height=400"` → `file test.png` возвращает `PNG image data`, а не текст-заглушку |
