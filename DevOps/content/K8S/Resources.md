---
tags:
  - kubernetes
  - resources
  - autoscaling
  - devops
aliases:
  - Kubernetes Resources
  - HPA
  - VPA
---

# Kubernetes Resources & Autoscaling

> [!summary] Главная модель
> **Requests** определяют, куда можно поставить Pod.  
> **Limits** ограничивают контейнер во время работы.  
> **HPA** меняет количество Pod.  
> **VPA** меняет размер одного Pod.  
> **Node Autoscaler** меняет количество Node.

---

# 1. Resources

## Requests и limits

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

| Поле | Назначение |
|---|---|
| `requests` | Scheduler использует при выборе Node; основа QoS, HPA и eviction |
| `limits` | Верхняя граница потребления контейнера |

Контейнер может потреблять больше `request`, пока не достигнет `limit`:

```text
request: 200m
usage:   350m  ← нормально
limit:   500m
```


---

## CPU и Memory ведут себя по-разному

```text
CPU usage > CPU limit
→ CPU throttling
→ контейнер работает медленнее

Memory usage > memory limit
→ возможен OOMKilled
→ процесс завершается, обычно exit code 137
```

```text
CPU    → сжимаемый ресурс
Memory → несжимаемый ресурс
```

---

## Scheduling

Scheduler размещает Pod по `requests`, а не по текущей загрузке Node.

```text
Node свободно по requests: 1 CPU
Новый Pod request:         2 CPU
↓
Pod остаётся Pending
```

Диагностика:

```bash
kubectl describe pod <pod>
kubectl describe node <node>
kubectl get events -A --sort-by='.lastTimestamp'
```

Типичное сообщение:

```text
0/1 nodes are available: 1 Insufficient cpu
```

---

## QoS-классы

| QoS | Условия |
|---|---|
| `Guaranteed` | У всех контейнеров CPU и memory requests равны limits |
| `Burstable` | Ресурсы заданы, но условия Guaranteed не выполнены |
| `BestEffort` | Requests и limits отсутствуют |

Проверка:

```bash
kubectl get pod <pod> -o jsonpath='{.status.qosClass}'
```

---

# 2. Metrics Server

Metrics Server предоставляет Kubernetes текущие CPU/memory-метрики:

```text
kubelet
↓
Metrics Server
↓
metrics.k8s.io
├── kubectl top
├── HPA
└── VPA
```

Команды:

```bash
kubectl top nodes
kubectl top pods -A
kubectl top pods -A --containers
```

```text
Metrics Server → текущие CPU/RAM для autoscaling
Prometheus     → история, графики и алерты
```

---

# 3. Horizontal Pod Autoscaler

> HPA отвечает: **сколько Pod должно работать?**

```text
Нагрузка выросла
↓
HPA увеличил replicas
↓
Deployment создал Pod
↓
Service распределил нагрузку
```

## Пример HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp

  minReplicas: 2
  maxReplicas: 10

  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15

    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
```

---

## Как HPA считает CPU

```text
CPU utilization = CPU usage / CPU request × 100%
```

Если:

```text
CPU request: 200m
Target:       50%
```

целевая нагрузка одного Pod:

```text
200m × 50% = 100m
```

> [!important]
> HPA сравнивает usage с **request**, а не с limit и не с размером Node.

Упрощённая формула:

```text
desiredReplicas =
ceil(currentReplicas × currentUtilization / targetUtilization)
```

Пример:

```text
Текущие replicas: 3
Текущий CPU:      100%
Target CPU:        50%

ceil(3 × 100 / 50) = 6 Pod
```

---

## ScaleUp и ScaleDown

### ScaleUp

```yaml
stabilizationWindowSeconds: 0
```

Реакция начинается без окна стабилизации.

```yaml
value: 100
periodSeconds: 15
```

Количество Pod может максимум удвоиться за соответствующий период:

```text
1 → 2 → 4 → 8 → 10
```

### ScaleDown

```yaml
stabilizationWindowSeconds: 300
```

HPA смотрит рекомендации за последние пять минут и выбирает наиболее осторожную. Это защищает от скачков:

```text
10 → 2 → 10
```

```yaml
value: 50
periodSeconds: 60
```

За минуту можно удалить максимум половину текущих Pod:

```text
10 → 5 → 2–3 → 1
```

`periodSeconds` задаёт максимальное изменение за период, а не гарантированный интервал масштабирования.

---

## Проверка HPA

```bash
kubectl get hpa
kubectl get hpa webapp-hpa --watch
kubectl describe hpa webapp-hpa
kubectl top pods -l app=webapp
```

Пример:

```text
TARGETS   MINPODS   MAXPODS   REPLICAS
80%/50%   2         10        5
```

```text
80% → текущая загрузка
50% → целевая загрузка
```

---

# 4. Vertical Pod Autoscaler

> VPA отвечает: **какого размера должен быть один Pod?**

```text
Request: 100m CPU
Usage:   800m CPU
↓
VPA рекомендует увеличить request
```

VPA устанавливается отдельно и состоит из:

```text
Recommender         → рассчитывает рекомендации
Updater             → применяет изменения/пересоздаёт Pod
Admission Controller → подставляет resources новым Pod
```

---

## Безопасный VPA

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: webapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp

  updatePolicy:
    updateMode: "Off"

  resourcePolicy:
    containerPolicies:
      - containerName: webapp
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: "2"
          memory: 2Gi
        controlledResources:
          - cpu
          - memory
        controlledValues: RequestsOnly
```

`Off` означает: VPA только показывает рекомендации и ничего автоматически не меняет.

Проверка:

```bash
kubectl get vpa
kubectl describe vpa webapp-vpa
```

Основные поля рекомендации:

```text
target       → рекомендуемый request
lowerBound   → нижняя граница
upperBound   → верхняя граница
uncappedTarget → рекомендация без min/max ограничений
```

---

## Режимы VPA

| Режим               | Поведение                                                     |
| ------------------- | ------------------------------------------------------------- |
| `Off`               | Только рекомендации                                           |
| `Initial`           | Применение ресурсов при создании Pod                          |
| `Recreate`          | VPA может удалить Pod, чтобы создать его с новыми ресурсами   |

Для первого production-внедрения:

```text
Off → анализ → ручное изменение → нагрузочный тест
```

Автоматический VPA опасно сразу включать для базы данных, одной replica и stateful workload.

---

## HPA и VPA вместе

Опасная комбинация:

```text
HPA масштабируется по CPU utilization
VPA одновременно меняет CPU request
```

Поскольку:

```text
utilization = usage / request
```

изменение request меняет метрику, по которой работает HPA.

Безопасные варианты:

```text
1. HPA по CPU + VPA в режиме Off
2. HPA по RPS/queue length + VPA по CPU/memory
3. HPA по CPU + VPA контролирует только memory
```

---

# 5. Общая архитектура autoscaling

```text
                    Нагрузка
                        ↓
       ┌────────────────┼────────────────┐
       ↓                ↓                ↓
      HPA              VPA       Node Autoscaler
«Сколько Pod?»   «Какой размер?»  «Сколько Node?»
```

Пример цепочки:

```text
Нагрузка выросла
↓
HPA запросил 10 replicas
↓
На Node не хватает места
↓
Node Autoscaler добавил Node
↓
Scheduler разместил Pod
↓
VPA уточнил requests
```

---

# 6. Итоговая шпаргалка

```text
requests.cpu/memory → scheduling, QoS, HPA, eviction
limits.cpu          → CPU throttling
limits.memory       → возможен OOMKilled

Metrics Server      → текущие CPU/RAM
HPA                 → количество Pod
VPA                 → requests/limits одного Pod
Node Autoscaler     → количество Node
```

Диагностический минимум:

```bash
kubectl top nodes
kubectl top pods -A
kubectl get hpa
kubectl describe hpa <hpa>
kubectl describe pod <pod>
kubectl describe node <node>
kubectl get events -A --sort-by='.lastTimestamp'
```
