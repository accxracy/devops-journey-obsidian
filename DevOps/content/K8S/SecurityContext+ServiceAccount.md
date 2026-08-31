---
tags:
  - kubernetes
  - security
  - rbac
  - serviceaccount
  - securitycontext
aliases:
  - Безопасность Pod
  - ServiceAccount
  - SecurityContext
---

# 🔐 SecurityContext и ServiceAccount

Обе сущности связаны с безопасностью Pod, но отвечают за разные вещи:

* **`securityContext`** — с какими системными ограничениями запускается процесс контейнера.
* **`ServiceAccount`** — от имени какой учётной записи Pod обращается к Kubernetes API.

```text
securityContext = ограничения процесса
ServiceAccount  = личность Pod внутри кластера
RBAC            = полномочия этой личности
NetworkPolicy   = разрешённые сетевые соединения
```

---
## 🛡️ SecurityContext

`securityContext` управляет безопасностью контейнера во время запуска: UID/GID, root-доступом, Linux capabilities, повышением привилегий, seccomp и файловой системой.

В Dockerfile можно задать пользователя по умолчанию:

```dockerfile
RUN adduser -D -u 10001 appuser
USER 10001
```

Но Kubernetes дополнительно может **проверить и принудительно применить** ограничения без пересборки образа:

```yaml
spec:
  containers:
    - name: backend
      image: backend:1.0
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
        seccompProfile:
          type: RuntimeDefault
```

### Главные параметры

* **`runAsNonRoot: true`** — запрещает запуск от root.
* **`runAsUser` / `runAsGroup`** — задают UID и GID процесса.
* **`allowPrivilegeEscalation: false`** — запрещает процессу получать дополнительные привилегии.
* **`capabilities.drop: [ALL]`** — удаляет дополнительные Linux capabilities.
* **`readOnlyRootFilesystem: true`** — запрещает запись в корневую файловую систему контейнера.
* **`seccompProfile: RuntimeDefault`** — фильтрует потенциально опасные системные вызовы.
* **`fsGroup`** — задаёт группу для доступа контейнеров к подключённым Volume.

> `securityContext` на уровне Pod применяется ко всем контейнерам, а на уровне конкретного контейнера — только к нему. Контейнерные настройки имеют приоритет при пересечении.

---
## 🪪 ServiceAccount

`ServiceAccount` — это **профиль приложения внутри Kubernetes**. API Server видит Pod примерно как:

```text
system:serviceaccount:<namespace>:<service-account>
```

Если `serviceAccountName` не указан, Pod использует ServiceAccount `default` своего namespace.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backend
  namespace: shopmonitor
---
apiVersion: v1
kind: Pod
metadata:
  name: backend
  namespace: shopmonitor
spec:
  serviceAccountName: backend
  containers:
    - name: backend
      image: backend:1.0
```

Сам ServiceAccount **не является ролью** и почти не выдаёт прав. Разрешения назначаются через RBAC:

```text
Pod → ServiceAccount → RoleBinding → Role → permissions
```

Пример разрешения читать Pod'ы:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: shopmonitor
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backend-pod-reader
  namespace: shopmonitor
subjects:
  - kind: ServiceAccount
    name: backend
    namespace: shopmonitor
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Проверка прав:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:shopmonitor:backend \
  -n shopmonitor
```

### Токен ServiceAccount

По умолчанию Kubernetes может примонтировать внутрь Pod:

```text
/var/run/secrets/kubernetes.io/serviceaccount/
├── ca.crt
├── namespace
└── token
```

Если приложению не нужен Kubernetes API, автоматическое подключение токена лучше отключить:

```yaml
spec:
  serviceAccountName: backend
  automountServiceAccountToken: false
```

---
## 🧠 Главное различие

| Механизм | На какой вопрос отвечает? |
|---|---|
| `securityContext` | Что процессу разрешено на уровне ОС и контейнера? |
| `ServiceAccount` | От имени кого Pod обращается к Kubernetes API? |
| `Role` / `ClusterRole` | Какие API-операции разрешены? |
| `RoleBinding` / `ClusterRoleBinding` | Кому назначены разрешения? |
| `NetworkPolicy` | Какие сетевые соединения разрешены? |

Короткая формула:

```text
ServiceAccount = кто ты
RBAC           = что тебе можно в Kubernetes API
SecurityContext = что может процесс внутри контейнера
NetworkPolicy   = куда может пройти сетевой трафик
```

