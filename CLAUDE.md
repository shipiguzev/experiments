# CLAUDE.md

Локальный k8s-стенд (kind) для PostgreSQL (Zalando `postgres-operator` + CloudNativePG) и ClickHouse
(Altinity `clickhouse-operator`) под VictoriaMetrics/Grafana/Loki. Полная структура репозитория и
порядок развёртывания — в README.md. Здесь только конвенции для внесения изменений.

## Технические конвенции

- **Правки — декларативные.** Проблемы синка оператора (permissions, owner, GRANT) чинить через
  манифест/values (например, имя владельца в CR), а не через ручной `kubectl exec`/`psql
  GRANT`/`ALTER`. Ручные правки теряются при следующем reconcile оператора.
- **Перед тем как утверждать, что расширение влияет на поведение — проверить `\dx`** в самой БД.
  Присутствие в `shared_preload_libraries` не означает, что расширение реально используется.
- **Helm v4:** голый `helm upgrade` переиспользует старые values по умолчанию. Использовать
  `--reset-values`, если явно не нужно смешивание со значениями предыдущего релиза.
- **`charts/postgres-cluster/values-production.yaml`** — самодостаточный файл текущего состояния
  кластера, не чейнить поверх `values-step*.yaml` (те моделируют историческую сборку и намеренно
  расходятся в паре мест — см. комментарий в начале файла).
- **Новый кластер ClickHouse** (по образцу `values-test2.yaml`/`values-test3.yaml`): отдельный
  namespace и `chiName`/`clusterName`, отдельный `backup.s3.pathPrefix`, плюс отдельная запись в
  `charts/monitoring-extras/values.yaml` под `clickhouseClusters:` (иначе не появится datasource
  в Grafana). См. skill `new-clickhouse-cluster`.
- **Новый кластер Postgres под Zalando-оператором** (по образцу `values-postgres2-pgexporter.yaml`):
  отдельный namespace, `clusterName`, `walgSecretStub: true` — мониторинг (`VMServiceScrape`)
  заводится автоматически из values чарта, отдельная запись в `monitoring-extras` не нужна (в
  отличие от ClickHouse). См. skill `new-postgres-cluster`.
- **Новый кластер под CNPG**: отдельный namespace/`clusterName`, отдельный `backup.destinationPath`
  внутри общего бакета и отдельный Secret с кредами (не темплейтится). Один оператор `cnpg-operator`
  общий для всех — `clusterWide: true`, per-кластерная конфигурация оператора не нужна. См. skill
  `new-cnpg-cluster`.
- **Major version upgrade Postgres** (Zalando/Spilo `inplace_upgrade.py`) — у скрипта нет
  dry-run: любой запуск при совместимых кластерах сразу выполняет реальный апгрейд данных. Патч CRD
  под новую версию откатывается при каждом рестарте пода оператора, не только при `helm upgrade`.
  См. skill `postgres-major-version-upgrade`.
- **WAL-G бэкапы не работают** (архив не пишется, `archive_command` = `/bin/true`, реплики не
  стартуют после апгрейда/пересоздания кластера) — оба известных класса багов и их причины
  см. skill `walg-backup-troubleshooting`.

## Конвенции доков (`docs/*.md`)

- Шаги нумеруются (`## Шаг N.` либо `## N.`) — не смешивать оба стиля в одном доке.
- Обязательный раздел `## Известные особенности` (не "Troubleshooting", не "Known issues") —
  реальные грабли, с которыми столкнулись при выполнении шагов, с причиной, а не только симптомом.
- При добавлении нового чарта/values-файла/кластера — обновить блок «Структура репозитория» в
  README.md в том же коммите/PR, что и сама правка (не отдельным follow-up).
- Скриншоты дашбордов лежат в `docs/images/*.png`, регенерируются через grafana-image-renderer —
  см. skill `dashboard-screenshots` и `docs/grafana-image-renderer-setup.md`.

## Git-конвенции

- Commit message: императив, с большой буквы, без точки в конце (`Add`, `Fix`, `Document`,
  `Rename` — не `Added`/`Fixed`).
- Одна логическая правка = один коммит. Если найденная в процессе работы граблина документируется
  отдельно от собственно фикса — отдельным коммитом (см. историю: `824cb81` добавляет кластер,
  `c2a6ee6` отдельно чинит дашборд под него).
