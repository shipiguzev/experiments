# Major version upgrade PostgreSQL 14 → 18 (Zalando postgres-operator)

Инструкция по выполнению major version upgrade PostgreSQL кластера под управлением Zalando postgres-operator с сохранением данных. Upgrade выполняется через встроенный механизм Spilo (`inplace_upgrade.py`) с downtime.

## Стартовая точка

- PostgreSQL 14 кластер запущен, 3 инстанса (1 Leader + 2 Replica)
- Zalando postgres-operator установлен с дефолтным образом `spilo-17:4.0-p3`
- Мониторинг настроен
- Данные в БД присутствуют
- Бекап через wal-g настроен

Проверьте текущее состояние:

```bash
kubectl get postgresql -n postgres
# NAME               TEAM   VERSION   PODS   STATUS
# postgres-cluster   test   14        3      Running

kubectl exec -it postgres-cluster-0 -n postgres -- psql -U postgres -c "SELECT version();"
# PostgreSQL 14.21
```

## Шаг 1. Обновить operator values и применить helm upgrade

Обновите `postgres/operator/postgres-operator-values.yaml` — добавьте новый образ Spilo и секцию `configMajorVersionUpgrade`:

```yaml
# postgres/operator/postgres-operator-values.yaml
configGeneral:
  docker_image: ghcr.io/zalando/spilo-18:4.1-p1

configMajorVersionUpgrade:
  major_version_upgrade_mode: "manual"
  minimal_major_version: "14"
  target_major_version: "18"
```

> **Важно:** параметры апгрейда должны быть в отдельной секции `configMajorVersionUpgrade`, а не в `configGeneral` — иначе `helm upgrade` завершится ошибкой `field not declared in schema`.

Примените:

```bash
helm upgrade postgres-operator postgres-operator-charts/postgres-operator \
  --namespace postgres-operator \
  --values postgres/operator/postgres-operator-values.yaml
```

Проверьте, что конфигурация применилась:

```bash
kubectl get operatorconfiguration postgres-operator -n postgres-operator \
  -o jsonpath='{.configuration}' | python3 -m json.tool | grep -i major
# "major_version_upgrade_mode": "manual",
# "minimal_major_version": "14",
# "target_major_version": "18"
```

## Шаг 2. Обновить CRD для поддержки версии 18

По умолчанию CRD оператора версии 1.15.1 поддерживает PostgreSQL только до версии 17. Версия 18 добавлена только в master-ветке. `helm upgrade` не обновляет CRD автоматически, а флаг `--include-crds` не поддерживается в текущей версии чарта — CRD нужно патчить вручную из master-ветки.

```bash
kubectl apply -f https://raw.githubusercontent.com/zalando/postgres-operator/master/charts/postgres-operator/crds/postgresqls.yaml
```

Проверьте, что версия 18 появилась в списке допустимых:

```bash
kubectl get crd postgresqls.acid.zalan.do \
  -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.postgresql.properties.version.enum}'
# Ожидаемый результат: ["14","15","16","17","18"]
```

> **Важно (грабли): патч CRD нужно переприменять не только после `helm upgrade`, а после КАЖДОГО (пере)старта пода оператора.** Сам оператор при старте выполняет `ensureCRDs` — ресинхронизирует CRD под свою прошитую (собранную в бинарник) схему, которая поддерживает только до версии 17, и тем самым откатывает наш патч. Это происходит не только на `helm upgrade`, но и на любом рестарте деплоймента `postgres-operator` — в том числе на приёме форсированного ресинка `kubectl rollout restart deployment postgres-operator`, который сами эти доки рекомендуют в нескольких местах ([`postgres-walg-backup-setup.md`](postgres-walg-backup-setup.md), раздел 6; troubleshooting `CreateFailed` в [`postgres-cluster-deployment-with-monitoring.md`](postgres-cluster-deployment-with-monitoring.md)). После любого такого рестарта проверяйте enum командой выше и при необходимости переприменяйте патч заново — иначе следующий `kubectl apply` манифеста с `version: "18"` упадёт с `Unsupported value: "18": supported values: "13","14","15","16","17"`.

## Шаг 3. Обновить версию в values чарта

Измените `postgresql.version` с `"14"` на `"18"` — это уже собрано отдельным overlay-файлом `charts/postgres-cluster/values-step4-v18.yaml`:

```yaml
postgresql:
  version: "18"   # было "14"
```

Примените — не забывая про `values-step3-walg.yaml` из предыдущего дока (`helm upgrade` не запоминает `-f` из прошлых вызовов):

```bash
helm upgrade postgres-cluster ./charts/postgres-cluster \
  --namespace postgres \
  --values charts/postgres-cluster/values.yaml \
  --values charts/postgres-cluster/values-step3-walg.yaml \
  --values charts/postgres-cluster/values-step4-v18.yaml
kubectl get postgresql -n postgres -w
# NAME               TEAM   VERSION   STATUS
# postgres-cluster   test   18        Updating
# postgres-cluster   test   18        Running
```

> **Важно:** оператор обновит spec и пересоздаст поды с образом `spilo-18` (образ подтянется автоматически), но реальный `pg_upgrade` данных не запустит. Версия сервера останется 14 — это нормально:
>
> ```bash
> kubectl exec -it postgres-cluster-0 -n postgres -- psql -U postgres -c "SELECT version();"
> # PostgreSQL 14.21  ← всё ещё 14, продолжаем
> ```

## Шаг 4. Определить текущий Leader

Upgrade запускается только на Leader-поде. Запуск на Replica завершится ошибкой `PostgreSQL is not running or in recovery`.

```bash
kubectl exec -it postgres-cluster-0 -n postgres -c postgres -- \
  bash -c "patronictl -c /run/postgres.yml list"
# + Cluster: postgres-cluster -------+----+
# | Member             | Role    | State     |
# | postgres-cluster-0 | Replica | streaming |
# | postgres-cluster-1 | Leader  | running   |  ← этот под
# | postgres-cluster-2 | Replica | streaming |
```

## Шаг 5. Запустить inplace_upgrade

Подключитесь к Leader-поду и запустите скрипт от пользователя `postgres`.

Аргумент — общее количество подов в кластере (не реплик): 3 пода → аргумент `3`.

> **Важно: у `inplace_upgrade.py` нет отдельного безопасного dry-run/check-only режима.** Скрипт всегда сам сначала выполняет `pg_upgrade --check`, и если кластеры совместимы — тут же, в рамках того же запуска и без дополнительного подтверждения, продолжает реальный апгрейд данных. Команда ниже (даже если отфильтровать её вывод через `grep -A20 "pg_upgrade --check"`, чтобы увидеть только результат проверки) — это не тестовый прогон, апгрейд фактически произойдёт.

```bash
kubectl exec -it postgres-cluster-1 -n postgres -c postgres -- \
  bash -c "su postgres -c 'python3 /scripts/inplace_upgrade.py 3 2>&1'"
```

Начало вывода покажет результат проверки совместимости:

```
Executing pg_upgrade --check
Checking cluster versions                                     ok
Checking database connection settings                         ok
Checking for prepared transactions                            ok
Checking data type usage                                      ok
Checking for not-null constraint inconsistencies              ok
Checking for presence of required libraries                   ok
*Clusters are compatible*
```

Дальше процесс продолжается автоматически и занимает ~18 секунд:

```
Cluster postgres-cluster is ready to be upgraded
initdb ... ok                          ← инициализация новой директории данных
pg_upgrade --check ... ok              ← проверка совместимости
Clusters are compatible
Doing a clean shutdown                 ← остановка кластера (начало downtime ~9 сек)
Executing pg_upgrade                   ← миграция данных через hard links
Upgrade Complete
Upgrade downtime: 9.28 seconds         ← конец downtime
Notifying replicas to start rsync      ← синхронизация реплик
Total upgrade time: 18.28 seconds
```

## Шаг 6. Проверка результата

Проверьте версию PostgreSQL:

```bash
kubectl exec -it postgres-cluster-1 -n postgres -- psql -U postgres -c "SELECT version();"
# PostgreSQL 18.2
```

Проверьте состояние кластера:

```bash
kubectl exec -it postgres-cluster-0 -n postgres -c postgres -- \
  bash -c "patronictl -c /run/postgres.yml list"
# | postgres-cluster-0 | Replica | streaming |  lag: 0
# | postgres-cluster-1 | Leader  | running   |
# | postgres-cluster-2 | Replica | streaming |  lag: 0
```

Проверьте целостность данных:

```bash
kubectl exec -it postgres-cluster-1 -n postgres -- \
  psql -U postgres -d app -c "SELECT count(*) FROM test_data;"
# count
# -----------
#  100000000  ← все данные на месте
```

> **Важно:** `patronictl list` сразу после `inplace_upgrade` может показывать не «streaming», а «start failed» у одной или обеих реплик — это не всегда норма. Если видите такую картину, переходите к Шагу 7, не считайте апгрейд завершённым.

## Шаг 7. Если реплики не стартуют после апгрейда (start failed)

**Симптом:**

```bash
kubectl exec -it postgres-cluster-1 -n postgres -c postgres -- \
  bash -c "patronictl -c /run/postgres.yml list"
# | postgres-cluster-0 | Replica | start failed | unknown |
# | postgres-cluster-1 | Leader  | running      |         |
# | postgres-cluster-2 | Replica | start failed | unknown |
```

Причём это повторяется даже после того, как Patroni сам пересоздаёт реплику через `reinitialize`/`bootstrapped from leader` — свежий базовый бэкап от текущего лидера падает с той же ошибкой.

**Диагностика** — смотрим лог postgres в упавшем поде:

```bash
kubectl logs -n postgres postgres-cluster-0 -c postgres --tail=100
# FATAL:  requested timeline 4 is not a child of this server's history
# DETAIL:  Latest checkpoint in file "pg_control" is at 3/D8000028 on timeline 1,
#          but in the history of the requested timeline, the server forked off
#          that timeline at 3/AC0000A0.
```

> Встречался и другой текст той же по сути ошибки — WAL-G не находит запрошенный файл истории/сегмент в архиве:
> ```
> ERROR: 2026/07/03 12:18:01 Archive '00000005.history' does not exist.
> ERROR: 2026/07/03 12:18:01 Archive '000000040000000000000018' does not exist.
> ```
> Причём `kubectl logs` может не показать эту ошибку вовсе — Spilo переключает вывод постмастера на `csvlog` в файл, а не в stdout контейнера. Смотреть реальный лог сервера нужно внутри пода:
> ```bash
> kubectl exec -n postgres postgres-cluster-0 -c postgres -- \
>   bash -c "ls /home/postgres/pgdata/pgroot/pg_log/ && tail -n 60 /home/postgres/pgdata/pgroot/pg_log/postgresql-*.log"
> ```

**Причина:** WAL-G-архив в MinIO/S3 (`WALG_S3_PREFIX`) общий и не привязан к конкретной инкарнации кластера/мажорной версии. `pg_upgrade` корректно создаёт новую директорию данных с чистой историей на timeline 1, но в бакете остаются WAL-сегменты и `*.history`-файлы от предыдущих таймлайнов старого PG14-кластера (переключения leader/replica, предыдущие тесты). Patroni по умолчанию восстанавливается с `recovery_target_timeline: latest`, находит в архиве более старший (протухший) `*.history`-файл и пытается накатываться по чужой, несовместимой истории — отсюда FATAL, причём он повторяется даже после полной реинициализации реплики, потому что протухший сам архив, а не локальные данные.

**Проверка гипотезы** — сверяем таймлайны в списке бэкапов (первые 8 hex-символов `wal_file_name` = таймлайн):

```bash
kubectl exec -n postgres postgres-cluster-1 -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env wal-g backup-list --detail"
# base_0000000200000003000000B3 ... pg_version 140018   ← старые PG14-таймлайны
# base_0000000300000003000000BB ... pg_version 140021
# base_0000000400000003000000C3 ... pg_version 140021
# base_0000000100000003000000D6 ... pg_version 180002   ← свежий PG18, timeline 1
```

Если видите смесь высоких таймлайнов от старой мажорной версии рядом со свежим бэкапом на timeline 1 — это тот самый баг.

**Решение** — очистить архив этого кластера (команда работает только в пределах `WALG_S3_PREFIX` данного кластера, других кластеров/бакетов не касается) и сразу снять новый базовый бэкап:

```bash
# 1. Полностью чистим протухший WAL/бэкап-архив кластера
kubectl exec -n postgres postgres-cluster-1 -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env wal-g delete everything --confirm"

# 2. Убеждаемся, что архив пуст
kubectl exec -n postgres postgres-cluster-1 -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env wal-g backup-list"
# No backups found

# 3. Снимаем свежий базовый бэкап на лидере через штатный скрипт Spilo
#    (напрямую wal-g backup-push использовать нельзя — коннект пойдёт от root
#    и упадёт с password authentication failed for user "root")
kubectl exec -n postgres postgres-cluster-1 -c postgres -- \
  bash -c "envdir /run/etc/wal-e.d/env su postgres -c '/scripts/postgres_backup.sh /home/postgres/pgdata/pgroot/data'"

# 4. Проверяем, что реплики сами поднялись (Patroni ретраит запуск раз в ~10-20 сек)
kubectl exec -n postgres postgres-cluster-1 -c postgres -- \
  bash -c "patronictl -c /run/postgres.yml list"
# | postgres-cluster-0 | Replica | streaming |  lag: 0
# | postgres-cluster-1 | Leader  | running   |
# | postgres-cluster-2 | Replica | streaming |  lag: 0
```

Специально пересоздавать реплики руками не нужно — как только архив перестаёт отдавать протухшую историю, Patroni сам успешно проходит recovery на очередной попытке.

## Известные особенности

| Проблема | Решение |
|---|---|
| CRD сбрасывается (enum версии откатывается на `13-17`) после `helm upgrade` или любого рестарта пода оператора (в т.ч. `kubectl rollout restart deployment postgres-operator`) | Оператор при старте выполняет `ensureCRDs` под свою прошитую схему — переприменять `kubectl apply -f` из master-ветки после каждого такого рестарта, не только после upgrade |
| `major_version_upgrade_mode` в `configGeneral` падает с ошибкой | Использовать отдельную секцию `configMajorVersionUpgrade` |
| Запуск `inplace_upgrade.py` без аргументов — пустой вывод | Передать количество подов: `python3 /scripts/inplace_upgrade.py 3` |
| Запуск от root — `initdb: error: cannot be run as root` | Запускать через `su postgres -c '...'` |
| Запуск на Replica — `PostgreSQL is not running or in recovery` | Определить Leader через `patronictl list` |
| `number of replicas does not match (3 != 2)` | Передавать общее количество подов (не реплик) |
| После `kubectl apply` версия сервера остаётся 14 | Нормально — реальный upgrade запускается через `inplace_upgrade.py` |
| Команда из Шага 5 воспринимается как безопасный тест | Отдельного dry-run режима у `inplace_upgrade.py` нет — любой его запуск при совместимых кластерах сразу выполняет реальный апгрейд данных |
| После `inplace_upgrade` реплики висят в `start failed`, в логе `FATAL: requested timeline N is not a child of this server's history` (или `ERROR: Archive '...' does not exist`) | Протух WAL-G архив (старые таймлайны прошлых инкарнаций кластера). Чистим: `wal-g delete everything --confirm`, затем снимаем новый бэкап через `/scripts/postgres_backup.sh` (см. Шаг 7) |
| `wal-g backup-push` напрямую падает с `password authentication failed for user "root"` | Использовать штатный `/scripts/postgres_backup.sh` от пользователя `postgres`, а не голый `wal-g backup-push` |
| Реинициализация реплики (`bootstrapped from leader`) не лечит `start failed` | Проблема не в локальных данных реплики, а в архиве — чинить нужно архив (Шаг 7), реинит один не поможет |
