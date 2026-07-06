# Установка pg_partman и партиционирование таблиц по дню

Инструкция по установке расширения `pg_partman` в PostgreSQL-кластер под управлением Zalando postgres-operator, созданию таблиц с нативным партиционированием по дню и генерации тестовых данных.

## Область применения

Предполагается, что уже развёрнуты:

- `postgres-operator` (helm-релиз `postgres-operator` в namespace `postgres-operator`);
- целевой Postgres-кластер (CR `postgresql.acid.zalan.do`) `postgres-cluster` в namespace `postgres`, версия 18;
- база `app` (создана через `spec.databases: {app: app_owner}`, владелец — роль `app_owner`, совпадающая по имени с той, что создаёт `preparedDatabases`).

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
    app: app_owner
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

> **Важно (грабли):** даже с объявленной схемой `partman` создание может упасть с тем же текстом `permission denied for database app`, но по другой причине — у роли-владельца схемы (`app_owner` при `defaultRoles: false`, либо `app_partman_owner` при `defaultRoles: true`) нет привилегии `CREATE` на самой базе `app`. Причина — рассинхронизация владельцев: `databases: {app: monitoring}` создаёт/держит базу `app` с владельцем `monitoring`, а `preparedDatabases` заводит для схем отдельную новую роль `app_owner`, которая владеет только схемой, но не самой базой (`CREATE` на базу неявно получает только её реальный владелец).
>
> Официальный путь миграции с `databases` на `preparedDatabases` ([docs/user.md](https://github.com/zalando/postgres-operator/blob/master/docs/user.md), раздел "From `databases` to `preparedDatabases`") — сделать так, чтобы имя владельца в `databases` совпадало с конвенцией `<dbname>_owner` из `preparedDatabases`, тогда это буквально одна и та же роль:
>
> ```yaml
> spec:
>   databases:
>     app: app_owner   # было: app: monitoring
> ```
>
> Роли синхронизируются раньше баз, поэтому при таком изменении оператор сам выполняет `ALTER DATABASE app OWNER TO app_owner;` (см. `executeAlterDatabaseOwner` в `pkg/cluster/database.go`) — `app_owner` становится настоящим владельцем базы и получает `CREATE` неявно, без ручного `GRANT`. Старая роль (`monitoring`), если она больше нигде не используется, может остаться в `users:` как есть — оператор не отбирает у неё существующие привилегии, она просто перестаёт быть владельцем `app`.
>
> Ручной `GRANT CREATE ON DATABASE app TO app_owner;` работает как временный обход, но это фактическая правка состояния БД в обход манифеста — при пересоздании кластера (новый `preparedDatabases`-овнер, новая база) грабли вернутся. Правка имени владельца в `databases:` устраняет проблему декларативно и навсегда.

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

Аналогично создаются остальные таблицы — в этом стенде это `public.metrics_daily` (колонка `ts`, метрика/значение) и `public.request_logs_daily` (колонка `ts`, лог HTTP-запросов):

```sql
CREATE TABLE public.metrics_daily (
    id          bigserial,
    ts          timestamptz NOT NULL DEFAULT now(),
    metric_name text NOT NULL,
    value       double precision NOT NULL,
    PRIMARY KEY (id, ts)
) PARTITION BY RANGE (ts);

SELECT partman.create_parent(
    p_parent_table     => 'public.metrics_daily',
    p_control          => 'ts',
    p_interval         => '1 day',
    p_premake          => 7,
    p_start_partition  => (current_date - 6)::text
);

CREATE TABLE public.request_logs_daily (
    id          bigserial,
    ts          timestamptz NOT NULL DEFAULT now(),
    method      text NOT NULL,
    path        text NOT NULL,
    status_code integer NOT NULL,
    duration_ms integer NOT NULL,
    PRIMARY KEY (id, ts)
) PARTITION BY RANGE (ts);

SELECT partman.create_parent(
    p_parent_table     => 'public.request_logs_daily',
    p_control          => 'ts',
    p_interval         => '1 day',
    p_premake          => 7,
    p_start_partition  => (current_date - 6)::text
);
```

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

Аналогичные `INSERT` выполняются для `metrics_daily` и `request_logs_daily`:

```sql
INSERT INTO public.metrics_daily (ts, metric_name, value)
SELECT
    d::date + (random() * interval '1 day'),
    (ARRAY['cpu_usage','memory_usage','disk_io','network_in','network_out'])[floor(random()*5+1)],
    round((random()*100)::numeric, 2)
FROM generate_series(current_date - 6, current_date, interval '1 day') d,
     generate_series(1, 2000) g;

INSERT INTO public.request_logs_daily (ts, method, path, status_code, duration_ms)
SELECT
    d::date + (random() * interval '1 day'),
    (ARRAY['GET','POST','PUT','DELETE'])[floor(random()*4+1)],
    (ARRAY['/api/users','/api/orders','/api/products','/health','/api/login'])[floor(random()*5+1)],
    (ARRAY[200,201,400,404,500])[floor(random()*5+1)],
    floor(random()*500+5)::int
FROM generate_series(current_date - 6, current_date, interval '1 day') d,
     generate_series(1, 2000) g;
```

## Шаг 4. Включить автообслуживание через pg_partman_bgw

Партиции из Шага 2 создаются заранее (`p_premake => 7`), но дальше их никто не продлевает и не подчищает — `partman.run_maintenance_proc()` вызывается только руками. Без периодического вызова партиции перестанут появляться за пределами исходного окна `premake`, а старые (`retention` не задан, `retention_keep_table = true`) будут копиться бесконечно.

`pg_partman` поставляется с собственным background worker'ом (`pg_partman_bgw`), который сам дёргает `run_maintenance_proc()` по расписанию — не нужно городить отдельный `pg_cron`/k8s CronJob.

> **Важно (грабли):** `pg_partman_bgw` и `pg_partman` — не одно и то же для `shared_preload_libraries`. `pg_partman` — чистое SQL/PLpgSQL-расширение, у него нет `.so`-файла (проверяется через `pg_config --pkglibdir` внутри пода Spilo — там лежит только `pg_partman_bgw.so`). Если вписать в `shared_preload_libraries` `pg_partman` вместо `pg_partman_bgw`, постмастер не найдёт файл библиотеки и не поднимется **ни на одном из подов кластера** после рестарта — это полный outage, а не safe rolling restart. В список нужно добавлять только `pg_partman_bgw`.

Включается через `spec.postgresql.parameters` в манифесте кластера:

```yaml
# postgres/installations/postgres-cluster.yaml
spec:
  postgresql:
    version: "18"
    parameters:
      shared_preload_libraries: "bg_mon,pg_stat_statements,pgextwlist,pg_auth_mon,set_user,pg_stat_kcache,pg_partman_bgw"
      pg_partman_bgw.interval: "3600"
      pg_partman_bgw.role: "app_owner"
      pg_partman_bgw.dbname: "app"
```

- `pg_partman_bgw.interval` — как часто (в секундах) воркер вызывает `run_maintenance_proc()`; `3600` — раз в час.
- `pg_partman_bgw.role` / `pg_partman_bgw.dbname` — от чьего имени и в какой базе выполнять обслуживание. Без них воркер стартует, но не знает, что обслуживать.

> **Важно (грабли):** `shared_preload_libraries` в Patroni — это непрозрачная строка, а не список, который патчится по элементам. Когда оператор шлёт `PATCH /config` в Patroni API с новым значением этого параметра, он **целиком заменяет** предыдущую строку, а не добавляет к ней один элемент. На живом кластере это значение обычно уже непустое — было выставлено один раз при первичном бутстрапе Spilo (`configure_spilo.py`) и с тех пор хранится как persisted-конфиг в DCS, в манифесте CR может вообще не упоминаться. Поэтому нельзя просто прописать `shared_preload_libraries: "pg_partman_bgw"` — это молча удалит из preload все остальные библиотеки, которые реально используются (мониторинг, статистика запросов и т.д.), без явной ошибки при рестарте. Перед изменением нужно сначала посмотреть текущее значение:
>
> ```bash
> kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -c "SHOW shared_preload_libraries;"
> ```
>
> и переносить в манифест полный список + добавляемый элемент, а не только его.

Примените манифест — параметр требует рестарта Postgres, оператор сам выполнит rolling restart (сначала реплики, затем лидер):

```bash
kubectl apply -f postgres/installations/postgres-cluster.yaml
kubectl get postgresql -n postgres postgres-cluster -w
# STATUS: Updating -> Running
```

Проверьте, что воркер поднялся:

```bash
kubectl exec -n postgres postgres-cluster-0 -c postgres -- bash -c "ps aux | grep 'pg_partman master'"
# postgres: postgres-cluster: pg_partman master background worker
kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -c "SELECT name, setting FROM pg_settings WHERE name LIKE 'pg_partman_bgw%';"
```

### Оставляйте в shared_preload_libraries только то, что реально используется

Раз уж приходится трогать `shared_preload_libraries` целиком, стоит сразу убрать оттуда всё лишнее, а не просто дописать новый элемент к тому, что было. Наличие библиотеки в списке ещё не значит, что она используется — проверяется через `CREATE EXTENSION`/`\dx` по всем базам, а не по факту присутствия в строке:

```bash
kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -c "SELECT datname FROM pg_database WHERE datistemplate = false;"
# для каждой базы:
kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d <db> -c "\dx"
```

На этом стенде так нашлись два лишних пункта:

- `timescaledb` — был в `shared_preload_libraries`, но `CREATE EXTENSION timescaledb` не выполнялся ни в одной базе (`app`, `postgres`, `template1`). Просто убран из списка.
- `pg_cron` — расширение было создано в базе `postgres` (`cron.database_name` по умолчанию), но `cron.job` пуст (0 джобов) — не использовалось. Убран из `shared_preload_libraries`, а сам объект расширения удалён отдельно:

  ```bash
  kubectl exec -n postgres postgres-cluster-1 -c postgres -- psql -U postgres -d postgres -c "DROP EXTENSION pg_cron CASCADE;"
  ```

  > **Важно (грабли):** `DROP EXTENSION pg_cron;` без `CASCADE` падает с `cannot drop extension pg_cron because other objects depend on it (function cron.schedule_in_database(text,text,text) depends on schema cron)` — это внутренняя особенность упаковки pg_cron (перегруженная сигнатура функции), а не признак реального использования извне. Перед `CASCADE` стоит убедиться через `pg_depend`, что зависимость действительно принадлежит только самому расширению и ничего постороннего не зацепит:
  >
  > ```sql
  > SELECT pg_describe_object(classid, objid, 0)
  > FROM pg_depend d
  > JOIN pg_extension e ON e.extname = 'pg_cron'
  > WHERE d.refobjid = e.oid AND d.deptype != 'e';
  > -- пусто = зависимостей снаружи расширения нет, CASCADE безопасен
  > ```

Итоговый список на этом стенде: `bg_mon, pg_stat_statements, pgextwlist, pg_auth_mon, set_user, pg_stat_kcache, pg_partman_bgw` — библиотеки, для которых либо есть реально созданное расширение с данными (`pg_stat_statements`, `pg_stat_kcache`, `set_user`, `pg_auth_mon`), либо активная конфигурация без отдельного extension-объекта (`bg_mon` — HTTP-статус на 8080, используется встроенным `/scripts/renice.sh`; `pgextwlist` — реально настроенный `extwlist.extensions`), плюс новый `pg_partman_bgw`.

## Проверка

| Проверка | Команда |
|---|---|
| Расширение установлено | `kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app -c "\dx"` |
| Список партиций таблицы | `kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app -c "\dt public.events_daily*"` |
| Строки по партициям | `kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -d app -c "SELECT tableoid::regclass, count(*) FROM public.events_daily GROUP BY 1 ORDER BY 1;"` |
| В `_default`-партиции пусто (данные не "провалились" мимо диапазона) | добавить в предыдущий запрос `WHERE tableoid::regclass::text = 'public.events_daily_default'` |
| Итоговые счётчики по всем таблицам | `SELECT 'events_daily', count(*) FROM public.events_daily UNION ALL SELECT 'metrics_daily', count(*) FROM public.metrics_daily UNION ALL SELECT 'request_logs_daily', count(*) FROM public.request_logs_daily;` |
| Воркер `pg_partman_bgw` запущен | `kubectl exec -n postgres postgres-cluster-0 -c postgres -- bash -c "ps aux \| grep 'pg_partman master'"` |
| GUC `pg_partman_bgw.*` применились | `kubectl exec -n postgres postgres-cluster-0 -c postgres -- psql -U postgres -c "SELECT name, setting FROM pg_settings WHERE name LIKE 'pg_partman_bgw%';"` |

## Известные особенности

| Проблема | Решение |
|---|---|
| `preparedDatabases` без явного `schemas` создаёт схему `data` с новой ролью-овнером без `CONNECT` на существующую базу → `STATUS: UpdateFailed` | Явно указать существующую схему (`schemas: {public: {defaultRoles: false, defaultUsers: false}}`), не давать оператору создавать схему/роли с нуля для уже существующей базы |
| После неудачной синхронизации остаются лишние роли (`<db>_data_owner`, `_reader`, `_writer`) | Удалить вручную (`DROP ROLE IF EXISTS ...`) перед повторным `kubectl apply` |
| `extensions: {pg_partman: partman}` без схемы `partman` в `schemas` — `create extension: pq: schema "partman" does not exist` | Оператор не создаёт схему для `extensions` сам — добавить `partman` в `schemas` (с `defaultRoles: false, defaultUsers: false`) рядом с `public` |
| Даже с объявленной схемой `partman` — `create database schema: pq: permission denied for database app` | Владелец в `databases:` не совпадает с ролью `<dbname>_owner`, которую использует `preparedDatabases` — привести к одному имени (`databases: {app: app_owner}`), оператор сам выполнит `ALTER DATABASE ... OWNER TO` при следующей синхронизации |
| `p_interval => 'daily'` — `Special partition interval values from old pg_partman versions ... no longer supported` | В pg_partman 5.x использовать нативные интервалы Postgres: `'1 day'`, `'1 week'`, `'1 month'` |
| `SELECT partman.run_maintenance_proc();` — `partman.run_maintenance_proc() is a procedure` | Вызывать через `CALL partman.run_maintenance_proc();` |
| Партиционируемая колонка не входит в PK/уникальный констрейнт | Для нативного партиционирования PK обязан включать колонку партиционирования: `PRIMARY KEY (id, event_time)`, а не `PRIMARY KEY (id)` |
| `shared_preload_libraries: "pg_partman"` вместо `pg_partman_bgw` — постмастер не стартует ни на одном поде | У `pg_partman` нет `.so`-файла (чистое SQL-расширение), в preload нужен только `pg_partman_bgw` |
| Изменение `shared_preload_libraries` через манифест тихо роняет остальные библиотеки | Это опаяемая строка, Patroni её не мёржит, а заменяет целиком — перед правкой смотреть текущее значение (`SHOW shared_preload_libraries;`) и переносить полный список + новый элемент |
| `DROP EXTENSION pg_cron;` — `cannot drop extension pg_cron because other objects depend on it` | Внутренняя особенность упаковки (перегруженная `cron.schedule_in_database`), не внешняя зависимость — проверить `pg_depend` и дропать через `CASCADE` |
