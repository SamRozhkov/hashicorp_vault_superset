# Airflow Via Argo CD

Ниже инструкция именно для текущего репозитория.

Airflow разворачивается:

- через `Argo CD Application`
- из официального Helm chart `apache/airflow`
- со значениями из [airflow/values.yaml](https://github.com/SamRozhkov/hashicorp_vault_superset/blob/main/airflow/values.yaml)

## Какие файлы используются

- Argo CD приложение: [argocd/airflow-application.yaml](https://github.com/SamRozhkov/hashicorp_vault_superset/blob/main/argocd/airflow-application.yaml)
- values для Helm chart: [airflow/values.yaml](https://github.com/SamRozhkov/hashicorp_vault_superset/blob/main/airflow/values.yaml)

Текущая схема `Application` использует `sources`:

1. Helm chart Airflow из `https://airflow.apache.org`
2. этот git-репозиторий как источник values через `$values/airflow/values.yaml`

## Предварительные условия

Перед запуском должны быть готовы:

- установленный `Argo CD`
- ingress controller `nginx`
- рабочий storage class для PVC
- доступ к Kubernetes cluster через `kubectl`

Проверить ingress controller:

```bash
kubectl get pods -A | grep ingress
```

Проверить Argo CD:

```bash
kubectl get pods -n argocd
```

## Как установить Airflow

1. Применить Argo CD application:

```bash
kubectl apply -f /Users/semen/k8s/argocd/airflow-application.yaml
```

2. Проверить, что приложение создано:

```bash
kubectl get application -n argocd airflow
```

3. Дождаться синхронизации в Argo CD UI или проверить из CLI:

```bash
kubectl get application -n argocd airflow -o yaml
```

Нужные признаки:

- `status.sync.status: Synced`
- `status.health.status: Healthy`

## Что создаёт chart

В namespace `airflow` должны появиться:

- `airflow-api-server`
- `airflow-scheduler`
- `airflow-worker`
- `airflow-triggerer`
- `airflow-postgresql`
- `airflow-redis`
- `airflow-statsd`

Проверка:

```bash
kubectl get pods -n airflow
kubectl get svc -n airflow
```

## Доступ к Airflow

В текущей конфигурации включён встроенный ingress Airflow chart:

- host: `airflow.local`
- ingress class: `nginx`

Это задаётся в [airflow/values.yaml](https://github.com/SamRozhkov/hashicorp_vault_superset/blob/main/airflow/values.yaml).

Проверить ingress:

```bash
kubectl get ingress -n airflow
```

Если `airflow.local` не открывается, добавь запись в `/etc/hosts`:

```text
172.28.0.3 airflow.local
```

Если IP ingress другой, возьми его из:

```bash
kubectl get ingress -n airflow
```

После этого UI должен открываться по адресу:

```text
http://airflow.local
```

## Как обновлять конфигурацию

Если меняешь [airflow/values.yaml](https://github.com/SamRozhkov/hashicorp_vault_superset/blob/main/airflow/values.yaml):

1. закоммить изменения
2. отправь их в `origin/main`
3. дождись `refresh/sync` в Argo CD

Проверка актуального revision:

```bash
kubectl get application -n argocd airflow -o yaml
```

Смотри:

- `status.sync.revisions`
- `status.operationState`

## Важные настройки для Argo CD

В текущем values уже включены настройки, без которых Airflow часто не стартует под Argo CD:

- `createUserJob.useHelmHooks: false`
- `createUserJob.applyCustomEnv: false`
- `migrateDatabaseJob.useHelmHooks: false`
- `migrateDatabaseJob.applyCustomEnv: false`
- `migrateDatabaseJob.jobAnnotations.argocd.argoproj.io/hook: Sync`

Если откатить эти настройки, Airflow может зависнуть на ожидании миграций БД.

## Troubleshooting

### 1. Pod'ы висят на `wait-for-airflow-migrations`

Проверить логи:

```bash
kubectl logs -n airflow <pod-name> -c wait-for-airflow-migrations --previous
```

Если там есть:

```text
There are still unapplied migrations
```

значит не отработал `migrateDatabaseJob` или Argo CD ещё применяет старый revision.

Что проверить:

```bash
kubectl get application -n argocd airflow -o yaml
kubectl get jobs -n airflow
```

### 2. `Application` в состоянии `OutOfSync`

Обычно это значит:

- новый commit уже есть в git
- Argo CD ещё не применил его

Нужно сделать `Sync` в Argo CD UI.

Если висит старая операция:

1. `Terminate operation`
2. потом заново `Sync`

### 3. Открывается не Airflow

Проверь:

- открываешь ли именно `http://airflow.local`
- существует ли ingress `airflow-ingress`
- не перехватывает ли другой ingress тот же host

Проверка:

```bash
kubectl get ingress -A -o wide
```

### 4. Ingress есть, но сайт не открывается

Проверь:

```bash
kubectl get ingress -n airflow -o yaml
kubectl get svc -n airflow
```

Для текущей конфигурации ingress должен смотреть на service:

- `airflow-api-server`

## Полезные команды

Статус приложения:

```bash
kubectl get application -n argocd airflow
```

Подробный статус:

```bash
kubectl get application -n argocd airflow -o yaml
```

Поды:

```bash
kubectl get pods -n airflow -o wide
```

Сервисы:

```bash
kubectl get svc -n airflow
```

Ingress:

```bash
kubectl get ingress -n airflow
```

События:

```bash
kubectl get events -n airflow --sort-by=.metadata.creationTimestamp
```
