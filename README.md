# experiments

Песочница для локального Kubernetes (kind): PostgreSQL (Zalando `postgres-operator`) и ClickHouse (Altinity `clickhouse-operator`) под мониторингом VictoriaMetrics + Grafana, с бэкапами через WAL-G и отработкой major version upgrade PostgreSQL 14 → 18.

Все манифесты, values-файлы и Helm-чарты, использованные в инструкциях ниже, лежат прямо в этом репозитории — доки не абстрактные, а описывают именно то, что здесь развёрнуто. Операторы и стек мониторинга ставятся через публичные Helm-чарты с values-файлами из репозитория; кластеры (CR `postgresql`/`ClickHouseInstallation`) и обвязка мониторинга вокруг них — через собственные чарты из `charts/`.

## Структура репозитория

```
experiments/
├── cluster/
│   └── kind-config.yaml                       # конфиг kind-кластера (1 control-plane + 3 worker)
├── monitoring/
│   ├── vm-values.yaml                         # values для victoria-metrics-k8s-stack
│   ├── loki-values.yaml                       # values для grafana/loki (SingleBinary, filesystem)
│   └── promtail-values.yaml                   # values для grafana/promtail (DaemonSet сбора логов)
├── postgres/
│   └── operator/
│       └── postgres-operator-values.yaml      # values postgres-operator (образ Spilo, WAL-G, major upgrade)
├── clickhouse/
│   └── operator/
│       └── clickhouse-operator-values.yaml
├── cnpg/
│   └── operator/
│       └── cnpg-operator-values.yaml          # values cnpg/cloudnative-pg — второй, независимый Postgres-оператор
├── charts/
│   ├── postgres-cluster/                      # CR postgresql (3 инстанса) + monitoring-обвязка
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── values-step2-base.yaml             # состояние после доки 2 (кластер + мониторинг, без бэкапов)
│   │   ├── values-step3-walg.yaml             # + WAL-G бэкапы (доки 3)
│   │   ├── values-step4-v18.yaml              # + после major upgrade 14 → 18 (доки 4)
│   │   ├── values-step6-partman.yaml          # + pg_partman партиционирование (доки 6)
│   │   ├── values-step7-walg-exporter.yaml    # + wal-g-exporter (доки 7)
│   │   ├── values-step8-partition-archive.yaml # CronJob: архив старых pg_partman партиций в S3
│   │   ├── values-production.yaml             # самодостаточный full-stack (см. Быстрый старт), не чейнить со values-step*
│   │   ├── values-postgres2-pgexporter.yaml   # независимый второй кластер с pg_exporter вместо postgres-exporter (доки 14)
│   │   └── templates/
│   ├── cnpg-cluster/                          # CR Cluster (CNPG) + ScheduledBackup + VMPodScrape
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   ├── clickhouse-cluster/                    # CR ClickHouseInstallation + clickhouse-backup sidecar
│   │   ├── Chart.yaml
│   │   ├── values.yaml                        # дефолты = chi-test
│   │   ├── values-step5-base.yaml             # состояние после доки 5, до бэкапов (доки 9 их включает)
│   │   ├── values-test2.yaml                  # оверрайд для второго кластера (namespace clickhouse-2)
│   │   ├── values-test3.yaml                  # третий кластер, версия закреплена на 23.3.2 (проверка совместимости дашбордов со старым CH)
│   │   └── templates/
│   └── monitoring-extras/                     # ClickHouse datasource, дашборды Grafana, grafana-image-renderer
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── dashboards/                        # *.json дашбордов Grafana
│       └── templates/
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
9. **[Регулярный бэкап ClickHouse в S3 через clickhouse-backup](docs/clickhouse-backup-setup.md)** — sidecar-контейнер `clickhouse-backup` в поде `chi-test`, персистентный volume для `/var/lib/clickhouse`, инкрементальный + полный бэкап по расписанию.
10. **[Архивирование старых партиций pg_partman в S3](docs/postgres-partition-archive-setup.md)** — `CronJob`, который выгружает (`pg_dump -Fc`) партиции старше retention в отдельный S3-бакет и только после подтверждённой загрузки отсоединяет их (`DETACH`, без `DROP`) через `partman.drop_partition_time`.
11. **[Логи: Loki + Promtail на kind](docs/loki-logging-setup.md)** — Grafana Loki (SingleBinary, filesystem storage) + Promtail (DaemonSet), логи всех подов кластера доступны через Grafana Explore и переживают удаление самих объектов `Job`/`Pod`.
12. **[Автоматический рестарт при изменении ConfigMap: Stakater Reloader](docs/reloader-setup.md)** — правка кастомных метрик `postgres-exporter` подхватывается сама, без ручного `kubectl delete pod`, через `podAnnotations` + repair-цикл оператора.
13. **[CloudNativePG: второй Postgres-оператор бок о бок с Zalando](docs/cnpg-cluster-setup.md)** — независимый оператор `Cluster` CRD в своём namespace, бэкапы в тот же MinIO под отдельным префиксом, мониторинг через `VMPodScrape` вместо `VMServiceScrape`, официальный дашборд `cloudnative-pg/grafana-dashboards`.
14. *(побочная ветка, не часть основной цепочки)* **[PostgreSQL мониторинг с pg_exporter (pgsty)](docs/postgres-monitoring-guide-pg-exporter.md)** — независимый второй кластер (`postgres2`) с `pgsty/pg_exporter` вместо `prometheus-postgres-exporter`, включая разбор проблемы multi-platform образов в kind.

Каждая инструкция содержит не только happy path, но и раздел «Известные особенности» / troubleshooting с реально встреченными проблемами и их причинами.

## Предварительные требования

- Docker
- [kind](https://kind.sigs.k8s.io/)
- kubectl
- Helm

## Важно

Все пароли и креды в манифестах (`admin`/`admin` для Grafana, `minioadmin`/`minioadmin` для MinIO и т.п.) — только для локального dev-стенда. Для прод-окружения использовать секреты и сгенерированные пароли.
