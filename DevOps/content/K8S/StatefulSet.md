---
tags:
 - kubernetes
 - devops
 - statefulset
 - database
aliases:
 - StatefulSet
 - Базы данных в K8s
---

# ☸️ Kubernetes: StatefulSet (Отказоустойчивые БД)

Мы разобрали `Deployment`. Он идеально подходит для запуска веб-серверов или ботов (**Stateless**-приложений), которым не нужно хранить состояние. 
Но если ты попытаешься запустить через Deployment кластер базы данных (например, PostgreSQL Master-Slave), всё развалится.

## 💥 Проблема Deployment для баз данных
1. **Случайные имена:** Поды получают имена вроде `db-7b5c8f...`. Если Под умрет, новый Под получит совершенно другое имя и IP. Slave-база не сможет найти Master-базу, потому что ее адрес изменился.
2. **Общий диск:** Если указать PVC в Deployment, все реплики (Поды) попытаются одновременно записать данные на один и тот же диск. Возникнет конфликт блокировок (Split-Brain), и данные повредятся.
3. **Хаос запуска:** Deployment запускает и убивает Поды параллельно. Для БД это фатально — Slave-база не должна запускаться раньше, чем полностью загрузится Master.

---

## 🏛 Решение: StatefulSet
**StatefulSet** — это контроллер Кубернетеса, созданный специально для приложений с состоянием (Stateful), таких как базы данных (PostgreSQL, Redis, MongoDB, Kafka).

Он дает 3 железобетонные гарантии:

### 1. Стабильные сетевые имена (Stable Network ID)
Поды больше не получают случайные хэши. Они получают строгие порядковые номера, начиная с нуля: `db-0`, `db-1`, `db-2`.
* `db-0` всегда будет Master-узлом.
* Если `db-0` умирает, Кубернетес поднимает новый Под с **точно таким же именем и DNS-адресом** (`db-0`). Остальная система даже не заметит подмены.

### 2. Строгий порядок (Ordered Deployment)
StatefulSet запускает Поды строго по очереди. 
* Сначала он запускает `db-0` и ждет, пока он перейдет в статус `Ready` (БД полностью инициализировалась).
* Только после этого он запускает `db-1`, затем `db-2`.
* При удалении (Scale Down) он убивает их в строгом обратном порядке (`db-2`, затем `db-1`).

### 3. Индивидуальные диски (VolumeClaimTemplates)
Это главная магия StatefulSet. Тебе больше не нужно создавать PVC руками. 
В манифесте StatefulSet ты пишешь **Шаблон Заявки (VolumeClaimTemplate)**. 
* Когда запускается `db-0`, StatefulSet сам создает для него отдельный диск `pvc-db-0`.
* Когда запускается `db-1`, он создает для него свой собственный диск `pvc-db-1`.
* Если `db-0` умрет и воскреснет, StatefulSet автоматически примонтирует к нему его старый диск `pvc-db-0`.

---

## 📄 Пример манифеста (statefulset.yaml)

 apiVersion: apps/v1
 kind: StatefulSet
 metadata:
 name: postgres
 spec:
 serviceName: "postgres-service" # Обязательно нужен Headless Service
 replicas: 3
 selector:
  matchLabels:
  app: postgres
 template:
  metadata:
  labels:
   app: postgres
  spec:
  containers:
  - name: postgres
   image: postgres:15
   volumeMounts:
   - name: db-data
    mountPath: /var/lib/postgresql/data
 
 # Магия: Фабрика индивидуальных дисков для каждого Пода
 volumeClaimTemplates:
 - metadata:
   name: db-data
  spec:
   accessModes: [ "ReadWriteOnce" ]
   resources:
   requests:
    storage: 10Gi