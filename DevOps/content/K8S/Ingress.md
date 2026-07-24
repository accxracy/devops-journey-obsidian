---
tags:
  - devops
  - kubernetes
  - networking
  - ingress
aliases:
  - Ingress
  - K8s Ingress
  - Ingress Controller
---
# Kubernetes Ingress


## 📝 Концепт
**Ingress** — ресурс Kubernetes (уровня L7), который управляет внешним доступом к внутренним `Service` кластера. Выступает в роли reverse proxy: маршрутизирует HTTP/HTTPS трафик по путям (path) и хостам (host), терминирует SSL/TLS и балансирует нагрузку.


**Жизненный цикл трафика:**
`Client` → `Ingress Controller` → `Service` (port) → `Pod` (targetPort/containerPort)

---

## ⚙️ Манифест (Host + Path + TLS + Default)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx 

  defaultBackend:
    service:
      name: default-backend-service
      port:
        number: 80

  tls:
    - hosts:
        - app.example.com
        - api.example.com
      secretName: example-tls-secret 

  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80 

    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80