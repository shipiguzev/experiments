# Мониторинг на VictoriaMetrics + Grafana в kind

Инструкция по развёртыванию мониторинга через VictoriaMetrics и Grafana в локальном Kubernetes кластере (kind).

## Предварительные требования

Перед началом убедитесь, что установлены:

- Docker
- [kind](https://kind.sigs.k8s.io/)
- kubectl
- Helm

## Структура файлов

```
experiments/
├── cluster/
│   └── kind-config.yaml
├── monitoring/
│   └── vm-values.yaml
```

## Шаг 1. Создание kind кластера

Создайте файл конфигурации кластера с одним control-plane и тремя worker нодами. Три worker ноды нужны, чтобы разместить три реплики на разных нодах — это имитирует production окружение.

```bash
mkdir -p cluster
```

```yaml
# cluster/kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

Создайте кластер:

```bash
kind create cluster --config cluster/kind-config.yaml
```

Ноды не сразу переходят в `Ready` — дождитесь этого состояния перед следующим шагом:

```bash
kubectl wait --for=condition=Ready nodes --all --timeout=120s
kubectl get nodes
```

## Шаг 2. Установка VictoriaMetrics + Grafana

`victoria-metrics-k8s-stack` — это Helm chart, который разворачивает полный стек мониторинга:

- **VMSingle** — хранилище метрик (аналог Prometheus)
- **VMAgent** — агент сбора метрик со всех подов кластера
- **VMAlert** — алертинг
- **Grafana** — визуализация

В values мы задаём:

- конкретную версию Grafana 11.1.0 для совместимости с production
- плагин `vertamedia-clickhouse-datasource` (Altinity plugin for ClickHouse) для подключения к ClickHouse напрямую и визуализации данных из `system.query_log`

```bash
mkdir -p monitoring
```

```yaml
# monitoring/vm-values.yaml
grafana:
  enabled: true
  adminPassword: admin
  image:
    tag: "11.1.0"
  plugins:
    - vertamedia-clickhouse-datasource
  env:
    GF_INSTALL_PLUGINS: "vertamedia-clickhouse-datasource 3.4.11"
```

Добавьте Helm репозиторий и установите стек:

```bash
helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo update
helm install vm vm/victoria-metrics-k8s-stack \
  --namespace monitoring \
  --create-namespace \
  --values monitoring/vm-values.yaml
```

Дождитесь готовности всех подов:

```bash
kubectl get pods -n monitoring -w
```

Ожидаемый набор подов: `vm-grafana`, `vmagent-*`, `vmalert-*`, `vmalertmanager-*`, `vmsingle-*`, `vm-victoria-metrics-operator`, `vm-kube-state-metrics`, `vm-prometheus-node-exporter` (по одному на ноду) и разовый Job `vm-victoria-metrics-k8s-stack-sync-job`. Все они должны быть в статусе `Running`, кроме sync-job — он завершается и переходит в `Completed` (`0/1`), это ожидаемо.

## Шаг 3. Доступ к Grafana

```bash
kubectl port-forward -n monitoring svc/vm-grafana 3000:80
```

Откройте [http://localhost:3000](http://localhost:3000)

- Login: `admin`
- Password: `admin`

Проверить, что сервис отвечает и плагин ClickHouse установлен, можно без браузера:

```bash
curl -s http://localhost:3000/api/health
curl -s -u admin:admin http://localhost:3000/api/plugins/vertamedia-clickhouse-datasource/settings
```

## Проверка

| Компонент | Команда |
|---|---|
| Ноды кластера | `kubectl get nodes` |
| Поды стека мониторинга | `kubectl get pods -n monitoring` |
| Health check Grafana | `curl -s http://localhost:3000/api/health` (после port-forward) |
| Таргеты VMAgent | `kubectl port-forward -n monitoring svc/vmagent-vm-victoria-metrics-k8s-stack 8429:8429` → http://localhost:8429/targets |
