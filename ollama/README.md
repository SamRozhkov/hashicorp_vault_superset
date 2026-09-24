# Ollama + Open WebUI

Манифесты разворачивают Ollama и Open WebUI в namespace `ollama`. Данные моделей,
пользователей и чатов хранятся в отдельных PersistentVolumeClaim.

## Установка

```bash
kubectl apply -k /Users/semen/k8s/ollama
```

После готовности pod откройте `https://open-webui.local`. Для локального кластера
добавьте IP ingress-контроллера в hosts:

```text
<INGRESS_IP> open-webui.local
```

Первого пользователя Open WebUI назначает администратором. Модели можно загрузить
в интерфейсе либо командой:

```bash
kubectl exec -n ollama deploy/ollama -- ollama pull llama3.2
```

## Настройка

- Для GPU добавьте к контейнеру `ollama` подходящий для кластера лимит GPU и
  установите NVIDIA device plugin.
- Если в кластере нет default StorageClass, укажите `storageClassName` в обоих PVC.
- Перед production-развёртыванием замените плавающие image tags `latest` и `main`
  на проверенные версии.
