## 1. Вывод команды kubectl get all -n task-manager
```
NAME                                       READY   STATUS    RESTARTS   AGE
pod/backend-deployment-6978cf459-g2wph     1/1     Running   0          14m
pod/backend-deployment-6978cf459-kjcg6     1/1     Running   0          14m
pod/backend-deployment-6978cf459-kp8c7     1/1     Running   0          14m
pod/frontend-deployment-64946449d6-gsd4k   1/1     Running   0          17m
pod/network-test                           1/1     Running   0          5m42s

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/backend-service    ClusterIP      10.108.95.84    <none>        8080/TCP       10m
service/frontend-service   LoadBalancer   10.99.141.111   <pending>     80:30080/TCP   10m

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/backend-deployment    3/3     3            3           17m
deployment.apps/frontend-deployment   1/1     1            1           17m

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/backend-deployment-6558495fd6    0         0         0       17m
replicaset.apps/backend-deployment-6978cf459     3         3         3       14m
replicaset.apps/frontend-deployment-64946449d6   1         1         1       17m
```

## Результат тестирования сетевой связности между сервисами
Проверка связи из пода network-test:
- Команда: wget -qO- http://frontend-service
- Команда: wget -qO- http://backend-service:8080

## Разница между Pod и Deployment
- **Pod:** Контейнер (или контейнер + обертка), не имеющий механизмов автоматического восстановления
- **Deployment:** Обертка над Pod, которая управляет его версиями, масштабированием и гарантирует, что в кластере всегда запущено нужное число реплик

## Объяснение назначения различных типов Service`ов
- **ClusterIP:** Внутренний IP кластера, виден только другим подам
- **NodePort:** Пробрасывает порты наружу, сервир становится доступным извне
- **LoadBalancer:** Конфигурируется с облачным провайдером