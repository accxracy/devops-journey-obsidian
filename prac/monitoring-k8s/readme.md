# Kubernetes Monitoring Lab

## Архитектура

```text
Browser
   │
   ▼
Ingress NGINX
   ├── grafana.test ─────► Grafana Service
   └── prometheus.test ──► Prometheus Service

Prometheus
   │
   ▼
ServiceMonitor ──► backend-service ──► backend Pods
```

## Структура проекта

```text
.
├── README.md
├── deployment.yaml
├── data.yaml
├── Service.yaml
├── ServiceMonitor.yaml
└── ingress.yaml
```

- `deployment.yaml` — Deployment backend-приложения;
- `data.yaml` — PV и PVC для постоянного хранения данных;
- `Service.yaml` — ClusterIP Service для backend-приложения;
- `ServiceMonitor.yaml` — конфигурация сбора метрик Prometheus;
- `ingress.yaml` — маршрутизация к Grafana и Prometheus.

## 1. Установка Prometheus и Grafana

Добавить Helm-репозиторий:

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
```

Установить `kube-prometheus-stack` в namespace `monitoring`:

```bash
helm upgrade --install prom \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

Проверить запуск компонентов:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Все основные Pod должны перейти в состояние `Running`.

## 2. Развёртывание приложения

Применить Kubernetes-манифесты:

```bash
kubectl apply -f data.yaml
kubectl apply -f deployment.yaml
kubectl apply -f Service.yaml
kubectl apply -f ServiceMonitor.yaml
kubectl apply -f ingress.yaml
```

Или применить все YAML-файлы из текущего каталога:

```bash
kubectl apply -f .
```

Проверить ресурсы:

```bash
kubectl get deploy,pods,svc,pvc,ingress -n monitoring
kubectl get pv
kubectl get servicemonitor -n monitoring
```

## 3. Настройка Ingress

Проверить его состояние:

```bash
kubectl get pods,svc -n ingress-nginx
```

Контроллер должен находиться в состоянии `Running`:

```text
ingress-nginx-controller   1/1   Running
```



## 4. Получение пароля Grafana

Логин по умолчанию:

```text
admin
```

Получить пароль:

```bash
kubectl get secret prom-grafana \
  -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 --decode

echo
```



## 5. Настройка ServiceMonitor

`ServiceMonitor` выбирает Service по его labels, а не по имени.

В `Service.yaml` должна присутствовать метка:

```yaml
metadata:
  name: backend-service
  namespace: monitoring
  labels:
    app: backend-service
```

Селектор Service при этом выбирает Pod приложения:

```yaml
spec:
  selector:
    app: backend
```

Метка Service должна совпадать с селектором `ServiceMonitor`:

```yaml
spec:
  selector:
    matchLabels:
      app: backend-service
```

Имя порта также должно совпадать. Если в `ServiceMonitor` указано:

```yaml
endpoints:
  - port: metrics
    path: /metrics
```

то порт Service должен называться `metrics`:

```yaml
ports:
  - name: metrics
    protocol: TCP
    port: 80
    targetPort: mainport
```

> Обычный образ `nginx:alpine` не предоставляет Prometheus-метрики по адресу `/metrics`. Для успешного сбора метрик приложение должно самостоятельно реализовать этот endpoint либо необходимо подключить отдельный exporter.

Проверить обнаружение target можно в Prometheus:

```text
http://prometheus.test/targets
```





