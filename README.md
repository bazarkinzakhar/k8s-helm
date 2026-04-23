# K8s Production-Ready Stack

Репозиторий содержит эталонную реализацию деплоя микросервисного приложения в Kubernetes с использованием Helm, SOPS и GitHub Actions. Проект спроектирован с учетом стандартов безопасности (Zero Trust Network), отказоустойчивости и автоматизации CI/CD

## Архитектура и Технологический стек

- Orchestration: Kubernetes 1.26+
- Templating: Helm 3
- Secret Management: Mozilla SOPS (PGP/KMS)
- Security: Network Policies (Default Deny), Non-root containers
- CI/CD: GitHub Actions (Linting, Security Scanning, Automated Deploy)

---

## Как пользоваться

### 1. Подготовка окружения

Убедитесь, что у вас установлены:

- helm (v3.0+)
- sops (для работы с секретами)
- kubectl сконфигурированный на ваш кластер

### 2. Деплой

Для базовой установки в пространство имен production:

```bash
helm upgrade --install k8s-app ./charts/k8s-app \
  --namespace production \
  --create-namespace \
  --values ./charts/k8s-app/values.yaml
```

---

## Как и куда писать свое

### Добавление новых манифестов

Все новые ресурсы (Ingress, PVC, ConfigMaps) добавляются строго в директорию charts/k8s-app/templates/

> Важно: Используйте конструкцию {{ include "k8s-app.fullname" . }} для соблюдения консистентности

### Изменение параметров

Все переменные окружения, теги образов и лимиты ресурсов вынесены в charts/k8s-app/values.yaml

- Resources: Отредактируйте секцию resources, если приложению требуется больше памяти
- ReplicaCount: Измените для масштабирования

---

## Как подключить свои секреты

Проект использует SOPS для шифрования секретов в Git

1. Создайте файл charts/k8s-app/secrets.enc.yaml
2. Зашифруйте его своим ключом:
   ```bash
   sops --encrypt --pgp <YOUR_FINGERPRINT> --inplace charts/k8s-app/secrets.enc.yaml
   ```
3. В пайплайне GitHub Actions убедитесь, что добавлен секрет SOPS_PGP_KEY для дешифровки при деплое

---

## Как обновлять и откатывать

### Обновление приложения

При пуше в ветку main запускается пайплайн, который:

1. Проверяет код линтером
2. Сканирует образ на уязвимости через Trivy
3. Проверяет манифесты на ошибки безопасности через Checkov
4. Выполняет helm upgrade

### Стратегия обновления

Используется RollingUpdate:

- maxUnavailable: 25%
- maxSurge: 25%
  Это гарантирует, что во время обновления всегда будет доступно минимум 75% подов

### Откат (Rollback)

Если деплой прошел неудачно:

```bash
helm rollback k8s-app <REVISION_NUMBER> -n production
```

---

## Отказоустойчивость и Безопасность

- Liveness & Readiness Probes: Контролируют состояние приложения. Если под зависнет, K8s перезапустит его автоматически
- Network Policies: Весь трафик между подами запрещен по умолчанию. Разрешены только необходимые соединения
- Resource Quotas: Установлены limits и requests, что предотвращает истощение ресурсов ноды
- Security Context: Контейнеры запускаются от имени non-root пользователя (UID 1000) для минимизации рисков при взломе приложения

---

## Версионность

- App Version: 1.0.0
- Chart Version: 0.1.0

Любые изменения в инфраструктуре должны сопровождаться инкрементом version в Chart.yaml
