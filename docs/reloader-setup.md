# Автоматический рестарт при изменении ConfigMap: Stakater Reloader

Инструкция по развёртыванию [Stakater Reloader](https://github.com/stakater/Reloader) — контроллера, который отслеживает изменения `ConfigMap`/`Secret` и сам инициирует rollout зависимых workload'ов. Решает конкретную проблему, задокументированную как известная особенность в [`postgres-walg-exporter-monitoring.md`](postgres-walg-exporter-monitoring.md) и [`postgres-partition-archive-setup.md`](postgres-partition-archive-setup.md): правка `configmap-exporter-queries.yaml` (кастомные метрики `postgres-exporter`, например `pg_partman_partitions`) не подхватывается на лету — раньше требовался ручной `kubectl delete pod` (реплики → лидер).

## Область применения

Предполагается, что уже развёрнут `postgres-cluster` (Zalando `postgres-operator`) — Reloader сам по себе универсален и не завязан на Postgres, но в этом репозитории применяется именно к нему.

## Структура файлов

```
experiments/
└── charts/
    └── postgres-cluster/
        ├── values.yaml                  # + podAnnotations: {reloader.stakater.com/auto: "true"}
        └── templates/
            └── postgresql.yaml          # + spec.podAnnotations passthrough
```

Отдельного values-файла для самого Reloader не заводили — как и для MinIO ([`postgres-walg-backup-setup.md`](postgres-walg-backup-setup.md), шаг 1), дефолтов чарта достаточно (`watchGlobally: true` уже покрывает все namespace).

## 1. Установка Reloader

```bash
helm repo add stakater https://stakater.github.io/stakater-charts
helm repo update stakater
helm upgrade --install reloader stakater/reloader --version 2.2.14 \
  --namespace reloader --create-namespace
```

```bash
kubectl get pods -n reloader
# reloader-reloader-xxxxxxxxxx-xxxxx   1/1   Running
```

## 2. Аннотация на стороне Postgres-кластера

> **Важно (грабли, проверено вживую):** правильный домен — **`reloader.stakater.com/auto`**, а не `reloader.stakater.io/auto`. Установленная версия чарта (`2.2.14`, app `v1.4.19`) на `.io`-вариант вообще никак не реагирует — ни рестарта, ни единой строки в логах Reloader. Переключение на `.com` решает это полностью, проверено на тестовом `Deployment`.

`postgresql.acid.zalan.do` CRD не даёт способа задать аннотацию на сам объект `StatefulSet` (только `podAnnotations`, `serviceAnnotations`, `masterServiceAnnotations` — ни одного generic top-level поля). Поэтому аннотация ставится через `spec.podAnnotations`, которую оператор кладёт в **pod template** результирующего `StatefulSet`, а не в его собственные `metadata.annotations`:

```yaml
# charts/postgres-cluster/values.yaml
podAnnotations:
  reloader.stakater.com/auto: "true"
```

```yaml
# charts/postgres-cluster/templates/postgresql.yaml (добавлено в spec.postgresql)
{{- if .Values.podAnnotations }}
  podAnnotations:
    {{- toYaml .Values.podAnnotations | nindent 4 }}
{{- end }}
```

> **Важно (проверено вживую, тестовым `Deployment`/`StatefulSet` в отдельном namespace):** Reloader реагирует на аннотацию `reloader.stakater.com/auto`, размещённую **либо** на верхнеуровневых `metadata.annotations` самого workload'а, **либо** только на `spec.template.metadata.annotations` (pod template) — оба варианта рабочие. Так как CRD Zalando-оператора даёт нам только второй вариант, отдельно уточнять/добиваться top-level аннотации на StatefulSet не потребовалось.

Применяется обычным `helm upgrade` (полная цепочка `values-step*.yaml`, как во всех остальных доках):

```bash
helm upgrade postgres-cluster ./charts/postgres-cluster \
  --namespace postgres \
  --values charts/postgres-cluster/values.yaml \
  --values charts/postgres-cluster/values-step3-walg.yaml \
  --values charts/postgres-cluster/values-step4-v18.yaml \
  --values charts/postgres-cluster/values-step6-partman.yaml \
  --values charts/postgres-cluster/values-step7-walg-exporter.yaml \
  --values charts/postgres-cluster/values-step8-partition-archive.yaml
```

## 3. Как это на самом деле работает с Zalando-оператором (важный нюанс)

Reloader — универсальный контроллер, рассчитанный на обычные `Deployment`/`StatefulSet` со стратегией `RollingUpdate`: он патчит `spec.template` (что бампает `updateRevision`), а дальше нативный контроллер Kubernetes сам раскатывает поды по одному.

**У StatefulSet'ов, которыми управляет Zalando `postgres-operator`, `updateStrategy.type: OnDelete`** (проверено: `kubectl get statefulset postgres-cluster -o jsonpath='{.spec.updateStrategy}'` → `{"type":"OnDelete"}`) — это осознанное решение оператора: рестарт кластера Patroni нельзя доверять наивному rolling update кontroller'а Kubernetes (порядок «сначала реплики, потом лидер» и проверка состояния через Patroni REST API критичны, иначе возможен split-brain). При `OnDelete` **сам Kubernetes ничего не делает** после патча `spec.template` — поды пересоздаются с новым шаблоном только когда их кто-то реально удалит.

Проверено вживую: после того как Reloader обнаружил изменение `postgresql-exporter-custom-queries` и запатчил `StatefulSet` (в логах Reloader — `Changes detected in 'postgresql-exporter-custom-queries' ... updated 'postgres-cluster' of type 'StatefulSet'`), поды **не перекатились сами** — `currentRevision`/`updateRevision` разошлись, но ни один под не был удалён.

Кто в итоге доводит дело до конца — сам `postgres-operator`, через свой периодический **repair-цикл**:

```bash
kubectl get operatorconfigurations -n postgres-operator -o yaml | grep -E "resync_period|repair_period"
# resync_period: 30m   — полная пересинхронизация всех кластеров
# repair_period: 5m    — более лёгкая проверка/починка дрейфа состояния
```

`repair_period` (по умолчанию **5 минут**) — это и есть механизм, который замечает несовпадение фактического `StatefulSet` (пропатченного Reloader'ом) с вычисленным желаемым состоянием и запускает свой обычный процесс пересоздания подов (реплики → лидер, через Patroni), тот же самый, что описан в [`postgres-walg-backup-setup.md`](postgres-walg-backup-setup.md#6-форсированный-полный-ресинк-оператора) для форс-ресинка после апгрейда конфига оператора.

**Итоговая цепочка** после правки `configmap-exporter-queries.yaml` + `helm upgrade`:
1. Reloader обнаруживает изменение `ConfigMap` (секунды) и патчит pod template `StatefulSet`.
2. В течение `repair_period` (до 5 минут) оператор сам замечает дрейф и оркестрирует Patroni-aware пересоздание подов.

Никакого ручного `kubectl delete pod` не требуется — но и не мгновенно, а с задержкой до 5 минут. Кто спешит проверить результат сразу — можно форсировать немедленный ресинк оператора (тот же приём, что и после смены его собственного конфига):

```bash
kubectl rollout restart deployment postgres-operator -n postgres-operator
kubectl rollout status deployment postgres-operator -n postgres-operator --timeout=60s
```

## 4. Проверка

```bash
# Аннотация реально попала в pod template StatefulSet'а
kubectl get statefulset postgres-cluster -n postgres -o jsonpath='{.spec.template.metadata.annotations}'
# {"reloader.stakater.com/auto":"true"}

# Внести реальное изменение в конфиг метрик (например, поправить description)
# и накатить helm upgrade — та же команда, что в разделе 2.

# Reloader отреагировал
kubectl logs -n reloader deploy/reloader-reloader --tail=5 | grep postgresql-exporter-custom-queries
# ... updated 'postgres-cluster' of type 'StatefulSet' in namespace 'postgres'

# В течение ~5 минут (или сразу после форс-ресинка из раздела 3) поды пересоздаются
kubectl get pods -n postgres -l cluster-name=postgres-cluster
# AGE у всех трёх подов должен обновиться
```

Реально проверено на этом стенде: после правки `pg_partman_partitions` в `configmap-exporter-queries.yaml` и `helm upgrade`, Reloader обнаружил изменение и запатчил `StatefulSet` в течение секунд; после форсированного ресинка оператора все три пода (`postgres-cluster-0/1/2`) пересоздались в правильном порядке без единой ручной команды `kubectl delete pod`.

## 5. Известные особенности

| Проблема | Решение |
|---|---|
| Аннотация `reloader.stakater.io/auto` — Reloader никак не реагирует, ни ошибки, ни строки в логе | Актуальный домен — `reloader.stakater.com/auto` (`.com`, не `.io`). Проверено эмпирически на версии чарта `2.2.14`/app `v1.4.19` |
| CRD `postgresql.acid.zalan.do` не даёт способа поставить аннотацию на сам объект `StatefulSet` | Используется `spec.podAnnotations` — кладётся оператором в pod template, чего Reloader'у достаточно (проверено: сработало и с top-level, и с pod-template-only размещением аннотации) |
| После правки `ConfigMap` и патча Reloader'ом `StatefulSet` поды не перекатываются сразу | У `StatefulSet` от `postgres-operator` `updateStrategy: OnDelete` — Kubernetes сам ничего не делает; фактический перекат выполняет `repair_period` (по умолчанию 5 минут) самого оператора. Не баг, а особенность архитектуры: пересоздание подов Patroni-кластера всегда должно идти через оператора, не через голый rolling update |
| Нужен результат немедленно, а не через до 5 минут | `kubectl rollout restart deployment postgres-operator -n postgres-operator` форсирует немедленный полный ресинк — тот же приём, что и после смены конфига самого оператора |

## 6. Дальнейшие шаги (не входят в эту инструкцию)

- Аналогичная аннотация для `clickhouse-cluster` (у `ClickHouseInstallation` CRD своя модель шаблонов подов — `templates.podTemplates`, поведение Reloader с ним отдельно не проверялось).
- `reloader.matchLabels`/`resourceLabelSelector` — сузить область действия Reloader до конкретных `ConfigMap` вместо `watchGlobally: true` по всем namespace, если в кластере появятся ConfigMap'ы, менять которые часто и не нужно перекатывать сервисы.
