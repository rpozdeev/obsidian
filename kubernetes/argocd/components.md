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

## ApplicationSet

Позволяет **автоматически создавать ArgoCD-приложения** из шаблонов. Он особенно полезен, если у тебя **много похожих приложений**, например:

- один Helm-чарт для разных окружений (dev, staging, prod)
- микросервисы с одинаковой структурой
- деплой в несколько кластеров
- деплой из всех папок в Git

### Основные идеи ApplicationSet

- ApplicationSet — это CRD (CustomResourceDefinition)
- Он **генерирует обычные Application-объекты**, используя **шаблон + генератор**
- Поддерживает генераторы:
    - List — статический список
    - Git — все директории/файлы из Git
    - Clusters — все подключённые к ArgoCD кластеры
    - Matrix, Merge — комбинирование источников
    - SCM — из GitHub/GitLab API (через pull request’ы, branches и т.д.)

#### Пример: деплой в два окружения (dev и prod)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-set
spec:
  generators:
    - list:
        elements:
          - env: dev
          - env: prod
  template:
    metadata:
      name: '{{env}}-myapp'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/your-repo.git
        targetRevision: HEAD
        path: apps/{{env}}
      destination:
        server: https://kubernetes.default.svc
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

Этот ApplicationSet создаст два приложения:
- dev-myapp из apps/dev
- prod-myapp из apps/prod

### Git Generator
#### Сценарий

Eсть такая структура в Git:
```
my-repo/
└── apps/
    ├── service-a/
    ├── service-b/
    └── service-c/
```

Каждая папка — это директория с kustomization.yaml, helm или просто манифестами. Мы хотим, чтобы ApplicationSet сам создавал по одному приложению на каждую такую папку.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: apps-from-git
spec:
  generators:
    - git:
        repoURL: https://github.com/your-org/your-repo.git
        revision: HEAD
        directories:
          - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/your-repo.git
        targetRevision: HEAD
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

#### Что происходит:

- git.directories.path: apps/* — ищет все подкаталоги в apps/
- {{path}} — путь до директории, например apps/service-a
- {{path.basename}} — имя папки, например service-a
- ApplicationSet создаст ArgoCD-приложение для каждой из папок.