---
tags:
  - devops
  - kubernetes
  - security
  - networking
aliases:
  - Network Policy
  - K8s Firewall
date: 2026-07-25
---
# Kubernetes Network Policy


## 📝 Концепт
**Network Policy** — это внутренний фаервол Kubernetes (L3/L4). Управляет тем, какие поды имеют право общаться друг с другом и с внешним миром. 

По умолчанию сеть в K8s **плоская (Default Allow)**: любой под может отправить запрос любому поду, даже в другой namespace (тестовая среда может случайно достучаться до продакшен БД).

Как только к поду применяется хотя бы одна Network Policy, он переходит в режим **Default Deny** (белый список). Разрешено только то, что явно прописано в правилах. Все остальные запросы дропаются.

> [!warning] Важно: Зависимость от CNI
> Сам Kubernetes не умеет блокировать трафик. Чтобы политики работали, их должен поддерживать сетевой плагин (CNI). 
> ✅ **Calico — работает.
> ❌ **Flannel** — проигнорирует манифест (K8s напишет `created`, но трафик блокироваться не будет).

## 🧭 Направления трафика
- **Ingress** — входящий трафик (*Кто может стучаться ко мне?*)
- **Egress** — исходящий трафик (*Куда я могу стучаться?*)

Фильтрация настраивается через 3 вида селекторов:
1. `podSelector` (поиск подов по лейблам)
2. `namespaceSelector` (поиск целых неймспейсов по лейблам)
3. `ipBlock` (конкретные IP / подсети CIDR)

## ⚙️ Типовой манифест (Защита базы данных)

Пример: База данных принимает трафик ТОЛЬКО от бэкенда по порту 5432, а сама не может инициировать запросы никуда.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-protection
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database # К кому применяем политику (вешаем "замок")
  
  policyTypes:
    - Ingress       # Контролируем только входящий трафик

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend # Кто имеет право зайти
      ports:
        - protocol: TCP
          port: 5432       # По какому порту
```

## 🚨 Tech Debt / Подводные камни (Грабли)

1. **Забытый DNS (Egress ловушка)**
   Если ты закрываешь `Egress` (исходящий трафик), под потеряет возможность резолвить адреса через CoreDNS. Обязательно нужно явно разрешать UDP 53 в `kube-system`:
   ```yaml
   egress:
     - to:
         - namespaceSelector:
             matchLabels:
               kubernetes.io/metadata.name: kube-system
       ports:
         - port: 53
           protocol: UDP
   ```

2. **Пустой `podSelector: {}`**
   Если оставить селектор пустым, политика применится **вообще ко всем подам** в текущем неймспейсе (идеально для создания правила `Deny All` на весь namespace).

3. **Отсутствие `policyTypes`**
   Всегда явно указывай массив `policyTypes: [Ingress, Egress]`. Если этого не сделать, контроллер попытается угадать намерения по наличию блоков `ingress/egress`, что часто приводит к неочевидному поведению.

4. **Существующие соединения**
   NetworkPolicy не ретроактивны. Если хакер уже установил TCP-сессию с БД, а ты применил политику — текущая сессия не разорвется.

## 🛠 Шпаргалка / CLI

```bash
# Посмотреть текущие политики:
kubectl get networkpolicies -A

# Проверить детали конкретной политики:
kubectl describe networkpolicy <name>

# Узнать лейблы подов (для написания селекторов):
kubectl get pods --show-labels

# Проверка связности (дебаг):
kubectl exec frontend-pod -- curl -m 3 backend-service:8080
kubectl exec backend-pod -- nc -zv database-service 5432
```