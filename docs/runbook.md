# Runbook: messager (dev и prod в Kubernetes)

## 1. Окружения и namespace

- Dev: `messager-dev`.
- Prod: `messager-prod`.
- Кластер: `minikube` с одной нодой `minikube`.

Для корректного размещения pod'ов используются label ноды.

**bash:**
```bash
kubectl label node minikube workload=system --overwrite
kubectl label node minikube workload=app --overwrite
```

**PowerShell:**
```powershell
kubectl label node minikube workload=system --overwrite
kubectl label node minikube workload=app --overwrite
```

`workload=system` используется для `postgres`, `workload=app` — для `user-service` и `message-service`.

## 2. Структура манифестов

- Базовые ресурсы расположены в `k8s/base`.
- Для окружений используются overlays:
  - `k8s/overlays/dev`
  - `k8s/overlays/prod`
- Namespace задаётся в `kustomization.yaml` overlay, а не в base-манифестах.
- Для prod используются patch-файлы на количество реплик сервисов.

## 3. Развёртывание dev

1. Создать namespace:

**bash:**
```bash
kubectl create namespace messager-dev || true
```

**PowerShell:**
```powershell
kubectl create namespace messager-dev 2>$null
```

2. Применить overlay:

**bash:**
```bash
kubectl apply -k k8s/overlays/dev
```

**PowerShell:**
```powershell
kubectl apply -k k8s/overlays/dev
```

3. Проверить состояние pod'ов:

**bash:**
```bash
kubectl get pods -n messager-dev -o wide
```

**PowerShell:**
```powershell
kubectl get pods -n messager-dev -o wide
```

Ожидаемое состояние:
- `frontend` — `Running`
- `bff` — `Running`
- `user-service` — `Running`
- `message-service` — `Running`
- `postgres` — `Running`
- `migrate-users` — `Completed`
- `migrate-messages` — `Completed`

4. Проверить логи backend-сервисов:

**bash:**
```bash
kubectl logs -n messager-dev deployment/user-service
kubectl logs -n messager-dev deployment/message-service
```

**PowerShell:**
```powershell
kubectl logs -n messager-dev deployment/user-service
kubectl logs -n messager-dev deployment/message-service
```

В логах `user-service` должны появляться успешные запросы `POST /api/v1/users`, `GET /api/v1/users`, `GET /api/v1/users/:id`.

В логах `message-service` должен быть старт HTTP-сервера и регистрация маршрутов `/api/v1/messages`, `/api/v1/conversations`, `/api/v1/files`.

## 4. Развёртывание prod

1. Создать namespace:

**bash:**
```bash
kubectl create namespace messager-prod || true
```

**PowerShell:**
```powershell
kubectl create namespace messager-prod 2>$null
```

2. Применить overlay:

**bash:**
```bash
kubectl apply -k k8s/overlays/prod
```

**PowerShell:**
```powershell
kubectl apply -k k8s/overlays/prod
```

3. Проверить состояние pod'ов:

**bash:**
```bash
kubectl get pods -n messager-prod -o wide
```

**PowerShell:**
```powershell
kubectl get pods -n messager-prod -o wide
```

Целевое состояние:
- `frontend` — `Running`
- `bff` — `Running`
- `user-service` — `Running`
- `message-service` — `Running`
- `postgres` — `Running`
- `migrate-users` — `Completed`
- `migrate-messages` — `Completed`

## 5. Типовые проблемы и устранение

### 5.1. Postgres в статусе Pending

Диагностика:

**bash:**
```bash
kubectl describe pod -n messager-prod <postgres-pod>
```

**PowerShell:**
```powershell
kubectl describe pod -n messager-prod <postgres-pod>
```

Если в `Events` присутствует сообщение о несовпадении `node affinity/selector`, необходимо добавить label:

**bash:**
```bash
kubectl label node minikube workload=system --overwrite
```

**PowerShell:**
```powershell
kubectl label node minikube workload=system --overwrite
```

После этого pod `postgres` будет успешно назначен на ноду.

### 5.2. Миграции и сервисы падают с `connection refused`

Диагностика:

**bash:**
```bash
kubectl logs -n messager-prod job/migrate-users
kubectl logs -n messager-prod job/migrate-messages
kubectl logs -n messager-prod <user-service-pod> --previous
kubectl logs -n messager-prod <message-service-pod> --previous
```

**PowerShell:**
```powershell
kubectl logs -n messager-prod job/migrate-users
kubectl logs -n messager-prod job/migrate-messages
kubectl logs -n messager-prod <user-service-pod> --previous
kubectl logs -n messager-prod <message-service-pod> --previous
```

Если в логах присутствует ошибка подключения к `postgres:5432`, это означает, что backend и миграции стартовали раньше готовности Postgres.

После запуска Postgres необходимо удалить Job'ы и повторно применить overlay:

**bash:**
```bash
kubectl delete job -n messager-prod migrate-users
kubectl delete job -n messager-prod migrate-messages
kubectl apply -k k8s/overlays/prod
kubectl rollout restart deployment/user-service -n messager-prod
kubectl rollout restart deployment/message-service -n messager-prod
```

**PowerShell:**
```powershell
kubectl delete job -n messager-prod migrate-users
kubectl delete job -n messager-prod migrate-messages
kubectl apply -k k8s/overlays/prod
kubectl rollout restart deployment/user-service -n messager-prod
kubectl rollout restart deployment/message-service -n messager-prod
```

### 5.3. Ошибка `database does not exist`

Если миграции завершаются ошибкой вида:

```text
FATAL: database "users_db" does not exist
FATAL: database "messages_db" does not exist
```

необходимо проверить DSN в `Secret` и привести их к использованию существующей базы `postgres`, если проект разворачивается в конфигурации с одной базой данных.

После изменения `Secret`:

**bash:**
```bash
kubectl apply -k k8s/overlays/prod
kubectl delete job -n messager-prod migrate-users migrate-messages
kubectl apply -k k8s/overlays/prod
```

**PowerShell:**
```powershell
kubectl apply -k k8s/overlays/prod
kubectl delete job -n messager-prod migrate-users migrate-messages
kubectl apply -k k8s/overlays/prod
```

### 5.4. `user-service` и `message-service` в статусе Pending

Если в `describe pod` присутствует сообщение:

```text
0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector
```

необходимо добавить label:

**bash:**
```bash
kubectl label node minikube workload=app --overwrite
```

**PowerShell:**
```powershell
kubectl label node minikube workload=app --overwrite
```

После этого пересоздать pod'ы сервисов:

**bash:**
```bash
kubectl scale deployment user-service -n messager-prod --replicas=0
kubectl scale deployment message-service -n messager-prod --replicas=0
kubectl scale deployment user-service -n messager-prod --replicas=1
kubectl scale deployment message-service -n messager-prod --replicas=1
```

**PowerShell:**
```powershell
kubectl scale deployment user-service -n messager-prod --replicas=0
kubectl scale deployment message-service -n messager-prod --replicas=0
kubectl scale deployment user-service -n messager-prod --replicas=1
kubectl scale deployment message-service -n messager-prod --replicas=1
```

### 5.5. CrashLoopBackOff у backend-сервисов

Для просмотра причины последнего падения используется:

**bash:**
```bash
kubectl logs -n messager-prod <pod-name> --previous
```

**PowerShell:**
```powershell
kubectl logs -n messager-prod <pod-name> --previous
```

Чаще всего причиной являются недоступность Postgres, отсутствие базы данных или запуск сервиса до завершения миграций.

## 6. Проверка работоспособности

### 6.1. Проверка dev

**bash:**
```bash
kubectl get pods -n messager-dev -o wide
kubectl logs -n messager-dev deployment/user-service
kubectl logs -n messager-dev deployment/message-service
```

**PowerShell:**
```powershell
kubectl get pods -n messager-dev -o wide
kubectl logs -n messager-dev deployment/user-service
kubectl logs -n messager-dev deployment/message-service
```

По результатам проверки в dev:
- `frontend`, `bff`, `user-service`, `message-service`, `postgres` находятся в статусе `Running`.
- `migrate-users` и `migrate-messages` завершены в статусе `Completed`.
- `user-service` обрабатывает регистрацию, поиск пользователя и получение пользователя по id.

### 6.2. Проверка prod

**bash:**
```bash
kubectl get pods -n messager-prod -o wide
kubectl logs -n messager-prod deployment/user-service
kubectl logs -n messager-prod deployment/message-service
```

**PowerShell:**
```powershell
kubectl get pods -n messager-prod -o wide
kubectl logs -n messager-prod deployment/user-service
kubectl logs -n messager-prod deployment/message-service
```

По результатам проверки в prod:
- `frontend`, `bff`, `user-service`, `message-service`, `postgres` находятся в статусе `Running`.
- `migrate-users` и `migrate-messages` завершены в статусе `Completed`.
- `message-service` успешно стартует и публикует HTTP API на порту `8082`.

## 7. Краткая команда для общей проверки

**bash:**
```bash
kubectl get pods -n messager-dev -o wide
kubectl get pods -n messager-prod -o wide
```

**PowerShell:**
```powershell
kubectl get pods -n messager-dev -o wide
kubectl get pods -n messager-prod -o wide
```

Исправное состояние системы:
- в `messager-dev` все основные pod'ы находятся в `Running`, миграции — в `Completed`;
- в `messager-prod` все основные pod'ы находятся в `Running`, миграции — в `Completed`.