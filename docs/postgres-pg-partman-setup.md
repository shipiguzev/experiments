# Установка pg_partman и партиционирование таблиц по дню

Инструкция по установке расширения `pg_partman` в PostgreSQL-кластер под управлением Zalando postgres-operator, созданию таблиц с нативным партиционированием по дню и генерации тестовых данных.

## Область применения

Предполагается, что уже развёрнуты:

- `postgres-operator` (helm-релиз `postgres-operator` в namespace `postgres-operator`);
- целевой Postgres-кластер (CR `postgresql.acid.zalan.do`) `postgres-cluster` в namespace `postgres`, версия 18;
- база `app` (создана через `spec.databases: {app: monitoring}`, владелец — пользователь `monitoring`).

`pg_partman` уже входит в образ Spilo — устанавливать бинарники отдельно не нужно, только включить расширение в конкретной базе.

## Структура файлов

```
experiments/
└── postgres/
    └── installations/
        └── postgres-cluster.yaml   # добавлена секция preparedDatabases
```

## Шаг 1. Задать расширение декларативно в манифесте кластера

У оператора есть отдельное поле `spec.preparedDatabases.<db>.extensions` — оператор сам выполняет `CREATE EXTENSION ... SCHEMA ...` при синхронизации, вручную заходить в базу не нужно. **Но** целевая схема расширения (`partman`) должна быть уже объявлена в `schemas` — сам оператор её для `extensions` не создаёт (см. грабли ниже).

```yaml
# postgres/installations/postgres-cluster.yaml
spec:
  databases:
    app: monitoring
  preparedDatabases:
    app:
      defaultUsers: false
      schemas:
        public:
          defaultRoles: false
          defaultUsers: false
        partman:
          defaultRoles: false
          defaultUsers: false
      extensions:
        pg_partman: partman
```

> **Важно (грабли):** если оставить `schemas` пустым, оператор по умолчанию создаёт для `preparedDatabases` новую схему `data` с отдельным овнером `app_data_owner`. У этой новой роли нет `CONNECT` на существующую базу `app` (база создана раньше через обычное поле `databases`, а не через `preparedDatabases` с нуля) — синхронизация падает:
>
> ```
> level=error msg="could not sync prepared databases: error(s) while syncing prepared databases:
> error(s) while syncing schemas of prepared databases: could not execute create database schema:
> pq: permission denied for database app"
> ```
>
> При этом `kubectl get postgresql` покажет `STATUS: UpdateFailed`, но сами поды кластера это не затрагивает — падает только шаг синхронизации данных, StatefulSet и Patroni продолжают работать штатно. Лечится явным указанием уже существующей схемы `public` вместо дефолтной `data` и отключением дефолтных ролей/юзеров (`defaultRoles: false`, `defaultUsers: false`) — тогда оператор просто устанавливает расширение в уже существующую схему, без побочных ролей.
>
> Если синхронизация уже успела создать роли `app_data_owner`/`app_data_reader`/`app_data_writer` до того, как вы поправили манифест — удалите их вручную перед повторным `kubectl apply`:
>
> ```bash
> kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -c "
> DROP ROLE IF EXISTS app_data_owner;
> DROP ROLE IF EXISTS app_data_reader;
> DROP ROLE IF EXISTS app_data_writer;
> "
> ```

> **Важно (грабли):** одного `extensions: {pg_partman: partman}` без соответствующей записи в `schemas` недостаточно — Postgres требует, чтобы схема, указанная в `CREATE EXTENSION ... SCHEMA ...`, уже существовала, а оператор для секции `extensions` схему сам не создаёт (в отличие от грабли с `data` выше, которая касается только дефолтной схемы всей `preparedDatabases`, а не схемы конкретного расширения). Без явного `schemas.partman` синхронизация падает:
>
> ```
> level=info msg="creating extension \"pg_partman\" schema \"partman\""
> level=error msg="could not sync prepared database: ... could not execute create extension: pq: schema \"partman\" does not exist"
> ```
>
> Лечится добавлением `partman` в `schemas` рядом с `public` (как в примере выше), с тем же `defaultRoles: false, defaultUsers: false` — иначе схему создаст роль вида `app_partman_owner`, которая упирается в следующую грабли.

> **Важно (грабли):** даже с объявленной схемой `partman` создание может упасть с тем же текстом `permission denied for database app`, но по другой причине — у роли-владельца схемы (`app_owner` при `defaultRoles: false`, либо `app_partman_owner` при `defaultRoles: true`) нет привилегии `CREATE` на самой базе `app`. Она отсутствует, потому что `app` создана через обычное поле `databases: {app: monitoring}` с владельцем `monitoring`, а не через `preparedDatabases` с нуля — во втором случае оператор сам выдал бы `CREATE` новой db-owner роли при создании базы. Лечится одноразовым grant'ом (выполнять на **primary**, реплики в read-only и просто откажут с `cannot execute ... in a read-only transaction`):
>
> ```bash
> kubectl exec -n postgres postgres-cluster-1 -c postgres -- psql -U postgres -d app -c \
>   "GRANT CREATE ON DATABASE app TO app_owner;"
> ```
>
> После grant'а оператор сам не переретраит синхронизацию — нужно вызвать новый reconcile, например полным ресинком оператора:
>
> ```bash
> kubectl rollout restart deployment postgres-operator -n postgres-operator
> kubectl rollout status deployment postgres-operator -n postgres-operator --timeout=60s
> ```

Примените манифест:

```bash
kubectl apply -f postgres/installations/postgres-cluster.yaml
kubectl get postgresql -n postgres postgres-cluster
# STATUS должен быть Running
```

Проверьте, что расширение установлено:

```bash
kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app -c "\dx"
# pg_partman | 5.4.1 | ... | partman | Extension to manage partitioned tables by time or ID
```

## Шаг 2. Создать таблицы с нативным партиционированием по дню

`pg_partman` 5.x работает только поверх нативного партиционирования PostgreSQL (`PARTITION BY RANGE`) — сначала создаётся обычная партиционированная таблица средствами самого Postgres, затем `partman.create_parent()` берёт её под управление: создаёт партиции по интервалу и настраивает автообслуживание.

Партиционируемая колонка обязана входить в первичный ключ.

```sql
CREATE TABLE public.events_daily (
    id          bigserial,
    event_time  timestamptz NOT NULL DEFAULT now(),
    event_type  text NOT NULL,
    payload     jsonb,
    PRIMARY KEY (id, event_time)
) PARTITION BY RANGE (event_time);

SELECT partman.create_parent(
    p_parent_table     => 'public.events_daily',
    p_control          => 'event_time',
    p_interval         => '1 day',
    p_premake          => 7,
    p_start_partition  => (current_date - 6)::text
);
```

- `p_control` — колонка, по которой партиционируем (`timestamptz`).
- `p_interval => '1 day'` — суточные партиции.
- `p_premake => 7` — заранее создать 7 партиций вперёд от текущей даты.
- `p_start_partition` — с какой даты начинать (без неё `create_parent` создаст партиции только от текущей даты вперёд, а не назад).

> **Важно (грабли):** значение `p_interval => 'daily'` (алиас из старых версий pg_partman) в 5.x больше не поддерживается:
>
> ```
> ERROR: Special partition interval values from old pg_partman versions (daily) are no longer
> supported. Please use a supported interval time value from core PostgreSQL
> ```
>
> Использовать нужно валидный `interval` из самого Postgres — `'1 day'`, `'1 week'`, `'1 month'` и т.д., а не текстовые алиасы вроде `daily`/`weekly`/`monthly`.

Аналогично создаются остальные таблицы — в этом стенде это `public.metrics_daily` (колонка `ts`, метрика/значение) и `public.request_logs_daily` (колонка `ts`, лог HTTP-запросов).

После создания всех parent-таблиц досоздайте партиции на нужный диапазон явным вызовом обслуживания:

```sql
CALL partman.run_maintenance_proc();
```

> **Важно (грабли):** `partman.run_maintenance_proc()` — это `PROCEDURE`, а не `FUNCTION`. Вызов через `SELECT partman.run_maintenance_proc();` падает с:
>
> ```
> ERROR:  partman.run_maintenance_proc() is a procedure
> HINT:  To call a procedure, use CALL.
> ```
>
> Нужно использовать `CALL`, а не `SELECT`.

Проверьте, что партиции создались на нужный диапазон дат:

```bash
kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app -c "\dt public.events_daily*"
```

Ожидаемый набор: `events_daily` (родитель), `events_daily_default`, и по одной партиции `events_daily_pYYYYMMDD` на каждый день диапазона `p_start_partition` … `текущая дата + p_premake` дней.

## Шаг 3. Сгенерировать данные за неделю

Данные вставляются напрямую `INSERT ... SELECT` с `generate_series` — по одному дню на каждую партицию, таймстемп внутри дня случайный:

```sql
INSERT INTO public.events_daily (event_time, event_type, payload)
SELECT
    d::date + (random() * interval '1 day'),
    (ARRAY['login','logout','click','purchase','error'])[floor(random()*5+1)],
    jsonb_build_object('user_id', floor(random()*10000)::int, 'session', md5(random()::text))
FROM generate_series(current_date - 6, current_date, interval '1 day') d,
     generate_series(1, 2000) g;
```

`generate_series(current_date - 6, current_date, interval '1 day')` даёт 7 дней (неделя, включая сегодня), `generate_series(1, 2000)` — по 2000 строк на день. PostgreSQL сам направит каждую строку в нужную партицию по значению `event_time` — партиции для этого диапазона уже должны существовать (см. Шаг 2).

Аналогичные `INSERT` выполняются для `metrics_daily` и `request_logs_daily`.

## Проверка

| Проверка | Команда |
|---|---|
| Расширение установлено | `kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app -c "\dx"` |
| Список партиций таблицы | `kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app -c "\dt public.events_daily*"` |
| Строки по партициям | `kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app -c "SELECT tableoid::regclass, count(*) FROM public.events_daily GROUP BY 1 ORDER BY 1;"` |
| В `_default`-партиции пусто (данные не "провалились" мимо диапазона) | добавить в предыдущий запрос `WHERE tableoid::regclass::text = 'public.events_daily_default'` |
| Итоговые счётчики по всем таблицам | `SELECT 'events_daily', count(*) FROM public.events_daily UNION ALL SELECT 'metrics_daily', count(*) FROM public.metrics_daily UNION ALL SELECT 'request_logs_daily', count(*) FROM public.request_logs_daily;` |

## Известные особенности

| Проблема | Решение |
|---|---|
| `preparedDatabases` без явного `schemas` создаёт схему `data` с новой ролью-овнером без `CONNECT` на существующую базу → `STATUS: UpdateFailed` | Явно указать существующую схему (`schemas: {public: {defaultRoles: false, defaultUsers: false}}`), не давать оператору создавать схему/роли с нуля для уже существующей базы |
| После неудачной синхронизации остаются лишние роли (`<db>_data_owner`, `_reader`, `_writer`) | Удалить вручную (`DROP ROLE IF EXISTS ...`) перед повторным `kubectl apply` |
| `extensions: {pg_partman: partman}` без схемы `partman` в `schemas` — `create extension: pq: schema "partman" does not exist` | Оператор не создаёт схему для `extensions` сам — добавить `partman` в `schemas` (с `defaultRoles: false, defaultUsers: false`) рядом с `public` |
| Даже с объявленной схемой `partman` — `create database schema: pq: permission denied for database app` | У db-owner роли (`app_owner`/`app_partman_owner`) нет `CREATE` на базе `app`, т.к. база создана через `databases:`, а не `preparedDatabases` с нуля — выдать вручную `GRANT CREATE ON DATABASE app TO app_owner;` на **primary**, затем форсировать ресинк (`kubectl rollout restart deployment postgres-operator -n postgres-operator`) |
| `p_interval => 'daily'` — `Special partition interval values from old pg_partman versions ... no longer supported` | В pg_partman 5.x использовать нативные интервалы Postgres: `'1 day'`, `'1 week'`, `'1 month'` |
| `SELECT partman.run_maintenance_proc();` — `partman.run_maintenance_proc() is a procedure` | Вызывать через `CALL partman.run_maintenance_proc();` |
| Партиционируемая колонка не входит в PK/уникальный констрейнт | Для нативного партиционирования PK обязан включать колонку партиционирования: `PRIMARY KEY (id, event_time)`, а не `PRIMARY KEY (id)` |
