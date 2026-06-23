# Vault Setup For Superset

Ниже инструкция именно под текущую схему в этом репозитории:

- `Superset` работает в Kubernetes
- секреты подставляются через `Bank-Vaults vault-env`
- используется auth path `kubernetes`
- используется role `superset`
- секреты лежат в `KV v2`

## Что должно быть настроено в Vault

1. Включен `KV v2` engine по пути `secret/`.
2. Включен Kubernetes auth method по пути `kubernetes`.
3. В Vault есть policy, которая разрешает читать нужные секреты.
4. В Vault есть role `superset`, привязанная к:
   - `serviceAccount`: `superset`
   - namespace: `superset`
   - policy: `superset-policy`

Текущая конфигурация Superset использует:

- путь секрета: `secret/data/superset`
- ключ: `secret_key`
- auth path: `kubernetes`
- role: `superset`

Это видно в [superset/values.yaml](https://github.com/SamRozhkov/hashicorp_vault_superset/blob/main/superset/values.yaml).

## Что выполнить в Vault

Если `secret/` еще не включен:

```bash
vault secrets enable -path=secret kv-v2
```

Если Kubernetes auth еще не включен:

```bash
vault auth enable kubernetes
```

Настройка Kubernetes auth:

```bash
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  token_reviewer_jwt="<TOKEN_REVIEWER_JWT>" \
  kubernetes_ca_cert="<K8S_CA_CERT>"
```

Обычно:

- `TOKEN_REVIEWER_JWT` берут из service account с правами `tokenreviews`
- `K8S_CA_CERT` берут из `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`

## Policy для Superset

Создай policy:

```hcl
path "secret/data/superset" {
  capabilities = ["read"]
}
```

Применить:

```bash
vault policy write superset-policy superset-policy.hcl
```

Если нужен доступ к нескольким секретам внутри папки:

```hcl
path "secret/data/superset/*" {
  capabilities = ["read"]
}

path "secret/data/superset" {
  capabilities = ["read"]
}
```

## Role для Superset

```bash
vault write auth/kubernetes/role/superset \
  bound_service_account_names="superset" \
  bound_service_account_namespaces="superset" \
  policies="superset-policy" \
  ttl="24h"
```

## Как добавить текущий секрет SECRET_KEY

Сейчас приложение ожидает:

- путь: `secret/data/superset`
- key: `secret_key`

Команда:

```bash
vault kv put secret/superset \
  secret_key='СЮДА_СЛОЖНЫЙ_СЕКРЕТ'
```

Проверка:

```bash
vault kv get secret/superset
```

## Как Superset читает этот секрет

В [superset/values.yaml](https://github.com/SamRozhkov/hashicorp_vault_superset/blob/main/superset/values.yaml) указано:

```yaml
extraEnv:
  SECRET_KEY: vault:secret/data/superset#secret_key
```

Это значит:

- `vault-env` читает путь `secret/data/superset`
- берет поле `secret_key`
- подставляет его в env переменную `SECRET_KEY`

В [superset/values.yaml](https://github.com/SamRozhkov/hashicorp_vault_superset/blob/main/superset/values.yaml) приложение уже читает значение из runtime env:

```python
SECRET_KEY = os.getenv("SECRET_KEY")
```

## Как добавлять новые секреты

Если нужен еще один ключ в том же секрете:

```bash
vault kv patch secret/superset \
  db_password='mypassword' \
  oauth_client_secret='my-oauth-secret'
```

Потом в `values.yaml` можно ссылаться так:

```yaml
extraEnv:
  SECRET_KEY: vault:secret/data/superset#secret_key
  DB_PASS: vault:secret/data/superset#db_password
  GOOGLE_SECRET: vault:secret/data/superset#oauth_client_secret
```

После изменения `values.yaml` нужен redeploy:

```bash
helm upgrade superset superset/superset -n superset -f /Users/semen/k8s/superset/values.yaml --version 0.15.5
```

## Если хранить секреты по разным путям

Можно так:

```bash
vault kv put secret/superset/app secret_key='...'
vault kv put secret/superset/db password='...'
```

Тогда ссылки будут:

```yaml
extraEnv:
  SECRET_KEY: vault:secret/data/superset/app#secret_key
  DB_PASS: vault:secret/data/superset/db#password
```

## Минимум, который нужен прямо сейчас

Для текущей конфигурации достаточно, чтобы в Vault было:

1. `kv-v2` на `secret/`
2. `auth/kubernetes`
3. policy с доступом к `secret/data/superset`
4. role `superset` для `serviceAccount=superset`, `namespace=superset`
5. секрет:

```bash
vault kv put secret/superset secret_key='...'
```

## Готовый блок команд для Vault

Ниже минимальный набор команд для текущего кластера. Перед запуском замени:

- `<TOKEN_REVIEWER_JWT>` на JWT service account с правом `tokenreviews`
- `<K8S_CA_CERT>` на CA сертификат Kubernetes API
- `<SUPERSET_SECRET_KEY>` на реальный секрет для Superset

### 1. Включить KV v2

Нужно один раз, если engine `secret/` еще не существует.

```bash
vault secrets enable -path=secret kv-v2
```

### 2. Включить Kubernetes auth

Нужно один раз, если auth method `kubernetes` еще не включен.

```bash
vault auth enable kubernetes
```

### 3. Настроить Kubernetes auth

Эта команда позволяет Vault доверять токенам из Kubernetes.

```bash
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  token_reviewer_jwt="<TOKEN_REVIEWER_JWT>" \
  kubernetes_ca_cert="<K8S_CA_CERT>"
```

### 4. Создать policy

Policy даст Superset право читать секрет `secret/superset`.

Сначала создай файл `superset-policy.hcl`:

```hcl
path "secret/data/superset" {
  capabilities = ["read"]
}
```

Потом примени его:

```bash
vault policy write superset-policy superset-policy.hcl
```

### 5. Создать role для service account Superset

Эта роль связывает Kubernetes service account `superset` из namespace `superset` с policy `superset-policy`.

```bash
vault write auth/kubernetes/role/superset \
  bound_service_account_names="superset" \
  bound_service_account_namespaces="superset" \
  policies="superset-policy" \
  ttl="24h"
```

### 6. Создать секрет для Superset

Именно это значение будет подставлено в env переменную `SECRET_KEY`.

```bash
vault kv put secret/superset \
  secret_key="<SUPERSET_SECRET_KEY>"
```

### 7. Проверить секрет

Проверка, что секрет действительно записан.

```bash
vault kv get secret/superset
```

## Итоговый блок без лишнего текста

Если все исходные значения уже подготовлены, можно выполнить почти подряд:

```bash
vault secrets enable -path=secret kv-v2
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  token_reviewer_jwt="<TOKEN_REVIEWER_JWT>" \
  kubernetes_ca_cert="<K8S_CA_CERT>"

cat > superset-policy.hcl <<'EOF'
path "secret/data/superset" {
  capabilities = ["read"]
}
EOF

vault policy write superset-policy superset-policy.hcl

vault write auth/kubernetes/role/superset \
  bound_service_account_names="superset" \
  bound_service_account_namespaces="superset" \
  policies="superset-policy" \
  ttl="24h"

vault kv put secret/superset \
  secret_key="<SUPERSET_SECRET_KEY>"

vault kv get secret/superset
```
