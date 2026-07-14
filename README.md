# experiments

Песочница для локального Kubernetes (kind): PostgreSQL (Zalando `postgres-operator`) и ClickHouse (Altinity `clickhouse-operator`) под мониторингом VictoriaMetrics + Grafana, с бэкапами через WAL-G и отработкой major version upgrade PostgreSQL 14 → 18.

Все манифесты и values-файлы, использованные в инструкциях ниже, лежат прямо в этом репозитории — доки не абстрактные, а описывают именно то, что здесь развёрнуто.

## Структура репозитория

```
experiments/
├── cluster/
│   └── kind-config.yaml                       # конфиг kind-кластера (1 control-plane + 3 worker)
├── monitoring/
│   ├── vm-values.yaml                         # values для victoria-metrics-k8s-stack
│   ├── grafana-image-renderer.yaml            # опциональный сервис рендеринга скриншотов дашбордов
│   ├── clickhouse-datasource-cm.yaml          # ClickHouse datasource для Grafana
│   └── dashboards/
│       ├── postgresql-cluster-overview.json
│       ├── Altinity_ClickHouse_Operator_dashboard.json
│       └── ClickHouse_Queries_dashboard.json
├── postgres/
│   ├── operator/
│   │   ├── postgres-operator-values.yaml      # values postgres-operator (образ Spilo, WAL-G, major upgrade)
│   │   └── postgres-pod-config.yaml           # общий ConfigMap параметров WAL-G
│   └── installations/
│       ├── postgres-cluster.yaml              # CR postgresql (3 инстанса)
│       └── monitoring/
│           ├── postgres-metrics-svc.yaml
│           └── vmservicescrape-postgres.yaml
├── clickhouse/
│   ├── operator/
│   │   └── clickhouse-operator-values.yaml
│   └── installations/
│       ├── chi-test.yaml                      # CR ClickHouseInstallation (1 шард, 2 реплики)
│       ├── chi-test-2.yaml                    # второй кластер в отдельном namespace (clickhouse-2)
│       └── monitoring/
│           └── vmservicescrape-clickhouse-operator.yaml
└── docs/                                      # пошаговые инструкции (см. ниже)
```

## Быстрый старт

Нужно развернуть PostgreSQL + MinIO + мониторинг + pg_partman одним проходом, сразу на v18, без исторического пути через апгрейд — см. **[Полное развёртывание с нуля: PostgreSQL 18 + MinIO + мониторинг](docs/postgres-full-stack-from-scratch.md)**. Это единая инструкция, собранная из доков 2, 3, 6 и 7 ниже, с пояснениями «что и зачем» на каждом шаге.

## Порядок развёртывания

Ниже — детальные доки, рассчитанные на последовательное применение (каждая следующая предполагает, что предыдущая уже выполнена) и на разбор конкретных граблей глубже, чем в быстром старте:

1. **[VictoriaMetrics + Grafana Monitoring Stack на kind](docs/victoriametrics-grafana-monitoring-on-kind.md)** — создание kind-кластера, установка `victoria-metrics-k8s-stack` (VMSingle, VMAgent, VMAlert, Grafana).
2. **[Развёртывание PostgreSQL кластера с мониторингом](docs/postgres-cluster-deployment-with-monitoring.md)** — Zalando `postgres-operator`, кластер из 3 инстансов с sidecar `prometheus-postgres-exporter`, дашборд в Grafana.
3. **[Настройка бэкапа PostgreSQL через WAL-G в MinIO/S3](docs/postgres-walg-backup-setup.md)** — MinIO как S3-совместимое хранилище, непрерывное WAL-архивирование и cron base backup.
4. **[Major version upgrade PostgreSQL 14 → 18](docs/postgres-major-version-upgrade-14-to-18.md)** — inplace-апгрейд через встроенный механизм Spilo, с разбором реальных граблей (протухший WAL-G архив после апгрейда и т.п.).
5. **[ClickHouse + мониторинг на kind](docs/clickhouse-monitoring-stack-on-kind.md)** — Altinity `clickhouse-operator`, кластер ClickHouse (1 шард, 2 реплики), datasource и дашборды в Grafana, второй кластер в отдельном namespace для проверки multi-cluster.
6. **[Установка pg_partman и партиционирование таблиц по дню](docs/postgres-pg-partman-setup.md)** — декларативная установка расширения через `preparedDatabases`, нативное партиционирование `PARTITION BY RANGE`, генерация тестовых данных.
7. **[Мониторинг бэкапов WAL-G через wal-g-exporter](docs/postgres-walg-exporter-monitoring.md)** — sidecar-экспортёр метрик base backup/WAL-архива, дашборд в Grafana.
8. *(опционально)* **[Скриншоты дашбордов через grafana-image-renderer](docs/grafana-image-renderer-setup.md)** — отдельный сервис рендеринга для визуальной проверки дашбордов без браузера.

Каждая инструкция содержит не только happy path, но и раздел «Известные особенности» / troubleshooting с реально встреченными проблемами и их причинами.

## Предварительные требования

- Docker
- [kind](https://kind.sigs.k8s.io/)
- kubectl
- Helm

## Важно

Все пароли и креды в манифестах (`admin`/`admin` для Grafana, `minioadmin`/`minioadmin` для MinIO и т.п.) — только для локального dev-стенда. Для прод-окружения использовать секреты и сгенерированные пароли.
