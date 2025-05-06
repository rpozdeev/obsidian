# Основные компоненты
| **Сущность**   | **Назначение**                                       |
| -------------- | ---------------------------------------------------- |
| Application    | Описывает одно приложение: где, что и как развернуть |
| AppProject     | Ограничивает доступ к ресурсам и Git репо            |
| ApplicationSet | Автоматически создаёт много Application’ов           |
## Application

**Application** — это объект, который описывает, **что именно разворачивать**, **откуда** (Git/Helm repo), и **куда** (namespace/кластер).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  source:
    repoURL: https://github.com/my-org/my-repo
    path: helm/my-app
    targetRevision: main
    helm:
      valueFiles:
        - values.yaml
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- **source** — откуда взять манифесты (Git repo, Helm chart и т.д.)
- **destination** — куда развернуть (кластер и namespace)
- **syncPolicy** — автоматическая или ручная синхронизация
- **project** — логическая группировка

## AppProject

**AppProject** позволяет задать:
- Какие Git-репозитории разрешены
- Какие namespace’ы и кластеры разрешены
- Какие ресурсы можно использовать
- Какие роли и доступы назначены

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  description: Project for Team A
  sourceRepos:
    - https://github.com/team-a/*
  destinations:
    - namespace: team-a
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

## **ApplicationSet**

Если есть **много одинаковых приложений** (например, для каждого клиента/окружения/региона), ApplicationSet позволяет генерировать Application’ы по шаблону.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-from-dirs
spec:
  generators:
    - git:
        repoURL: https://github.com/my-org/apps
        revision: main
        directories:
          - path: "environments/*"
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/my-org/apps
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## Автоматическая синхронизация (syncPolicy)

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

Это включает **автоматическое обновление приложений** из Git при изменении репозитория.


## Lifecycle Hooks (хуки синхронизации)

Хуки позволяют выполнять дополнительные действия до/после синхронизации. Например:

- прогрев кеша
- миграции базы
- уведомления

Пример Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-sync-job
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: wait
        image: alpine
        command: ["sh", "-c", "echo Подготовка... && sleep 5"]
      restartPolicy: Never
```
### **Возможные типы хуков:**

- PreSync — до применения изменений
- Sync — замена основного ресурса (если надо)
- PostSync — после применения
- SyncFail — при ошибке синхронизации
### **Параметры удаления:**

- HookSucceeded
- HookFailed
- BeforeHookCreation (удалить перед созданием нового)