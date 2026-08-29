# K8s: RBAC (Role-Based Access Control)

**Теги:** `#kubernetes` `#rbac` `#security` `#devops`


**RBAC** — это базовая система контроля доступа в Kubernetes. Она определяет, **кто** (субъект), **что** (действие) и с **какими ресурсами** может делать. 

---

## 1. Базовые понятия (Из чего состоит RBAC)

Чтобы дать доступ, K8s связывает три компонента:
1. **Subject (Субъект)** — *кому* даем доступ.
2. **API Resource (Ресурс)** — *к чему* даем доступ.
3. **Verb (Действие)** — *что* разрешаем делать.

### Субъекты (Subjects)
- **User (Пользователь)** — человек (например, админ или разработчик). Управляется вне кластера
- **Group (Группа)** — набор пользователей (например, `devs`, `system:masters`).
- **ServiceAccount (SA)** — аккаунт для **приложений/Pod'ов** внутри кластера. K8s управляет ими сам. 

### Действия (Verbs)
Базовые операции с ресурсами:
- `get` (получить один манифест)
- `list` (получить список)
- `watch` (следить за изменениями в реальном времени)
- `create`, `update`, `patch`, `delete`.

---

## 2. Роли и Привязки (Roles & Bindings)

K8s разделяет правила и их назначение. Правила описываются в Ролях, а назначаются через Привязки (Bindings).

| Область действия | Сущность с правилами (Что можно?) | Сущность привязки (Кому можно?) |
| :--- | :--- | :--- |
| **Namespace** (Один проект) | `Role` | `RoleBinding` |
| **Cluster** (Весь кластер) | `ClusterRole` | `ClusterRoleBinding` |

*Важно:* `RoleBinding` может ссылаться на `ClusterRole`. Это полезно, если у тебя есть общая роль (например, `view-only`), и ты хочешь выдать её пользователю, но **только в одном namespace**.

---

## 3. Примеры манифестов

### Пример 1: Role (Даем права на чтение Pod'ов в namespace `backend`)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: backend
  name: pod-reader
rules:
- apiGroups: [""] # Пустая строка означает Core API (там живут Pods, Services)
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
```

### Пример 2: RoleBinding (Привязываем роль к ServiceAccount)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: backend
subjects:
- kind: ServiceAccount
  name: my-app-sa     # Имя ServiceAccount, которому даем права
  namespace: backend
roleRef:
  kind: Role          # Какую роль привязываем
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## 4. Как дебажить и проверять права? (must-know)

В K8s есть встроенная утилита `auth can-i` для проверки доступов. Отличная штука для траблшутинга.

Проверить свои права:
```bash
# Могу ли я создавать деплойменты в неймспейсе default?
kubectl auth can-i create deployments --namespace default
```

Проверить права конкретного ServiceAccount (очень полезно, когда Pod падает с ошибкой 403 Forbidden):
```bash
kubectl auth can-i list secrets \
  --as=system:serviceaccount:backend:my-app-sa \
  --namespace backend
```


