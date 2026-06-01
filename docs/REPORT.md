# Отчёт о проделанной работе

## Цель работы

Цель работы заключалась в развёртывании приложения messager в Kubernetes и настройке GitOps-деплоя через Argo CD, чтобы изменения, вносимые в GitHub-репозиторий, автоматически применялись в кластере без ручного использования `kubectl apply`.

## Развёртывание приложения в Kubernetes

В рамках выполнения работы были подготовлены Kubernetes-манифесты для приложения messager. Конфигурация приложения была организована с использованием Kustomize: выделен базовый набор манифестов и созданы отдельные overlays для окружений разработки и продакшн.

Структура конфигурации включала:

- базовые манифесты приложения;
- overlay для dev-окружения — `k8s/overlays/dev`;
- overlay для prod-окружения — `k8s/overlays/prod`.

Такой подход позволил использовать единый набор ресурсов приложения и при этом задавать различия между окружениями на уровне overlays.


## Установка и настройка Argo CD

Для реализации GitOps-подхода в кластер Kubernetes был установлен Argo CD. После установки был настроен доступ к веб-интерфейсу Argo CD с локальной машины с помощью проброса порта от сервиса `argocd-server` на `localhost`.

Пример команды для доступа к интерфейсу:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

После выполнения этой команды интерфейс Argo CD становился доступен по адресу `https://localhost:8080`.

Для входа в интерфейс использовалась стандартная учётная запись `admin`, а пароль извлекался из секрета `argocd-initial-admin-secret`, созданного при установке Argo CD.

## Подключение GitHub-репозитория

В качестве источника конфигурации использовался GitHub-репозиторий, содержащий Kubernetes-манифесты приложения. Argo CD был настроен на чтение манифестов из конкретной ветки репозитория и из указанных путей с overlays для соответствующих окружений.

GitHub-репозиторий выступал источником желаемого состояния, а Argo CD обеспечивал приведение состояния кластера к содержимому репозитория.

## Настройка приложения Argo CD для dev

Для окружения разработки было создано приложение `messager-dev`, связанное с overlay `k8s/overlays/dev`. Это приложение отвечает за развёртывание dev-версии messager в Kubernetes.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: messager-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-org>/<your-repo>.git
    targetRevision: main
    path: k8s/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: messager-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

В манифесте задавались репозиторий, ветка, путь к overlay и параметры автоматической синхронизации, включая `prune` и `selfHeal`.

## Настройка приложения Argo CD для prod

Для продакшн-окружения было создано отдельное приложение `messager-prod`, использующее overlay `k8s/overlays/prod`. Это позволило разворачивать prod-версию приложения независимо от dev-окружения.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: messager-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-org>/<your-repo>.git
    targetRevision: main
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: messager-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Разделение на два приложения позволило поддерживать два независимых окружения в рамках единого подхода GitOps.

## Проверка статуса приложений

После применения манифестов в Argo CD были зарегистрированы два приложения: `messager-dev` и `messager-prod`. Оба приложения успешно синхронизировались и перешли в состояние `Synced` и `Healthy`.


Для дополнительной проверки использовалась команда:

```bash
kubectl get applications -n argocd
```

Эта команда позволяла убедиться в наличии обоих приложений и их корректном состоянии в кластере.

## Проверка GitOps-процесса

После настройки Argo CD была подтверждена корректная работа GitOps-подхода. При изменении конфигурации приложения в GitHub-репозитории Argo CD автоматически обнаруживал изменения и синхронизировал кластер с актуальной версией манифестов.

Благодаря параметру `prune` Argo CD удалял ресурсы, которые были удалены из репозитория, а параметр `selfHeal` обеспечивал автоматическое восстановление корректного состояния при расхождении между кластером и Git-репозиторием.

Это обеспечило прозрачный, воспроизводимый и удобный процесс деплоя приложения в Kubernetes.

## Вывод

В результате проделанной работы приложение messager было успешно развёрнуто в Kubernetes, а для управления деплоем был настроен Argo CD. Были созданы отдельные приложения `messager-dev` и `messager-prod`, подключён GitHub-репозиторий и включена автоматическая синхронизация с параметрами `automated`, `prune` и `selfHeal`.

Настроенный процесс обеспечил автоматический деплой изменений из GitHub в Kubernetes и подтвердил работоспособность GitOps-подхода на практике.