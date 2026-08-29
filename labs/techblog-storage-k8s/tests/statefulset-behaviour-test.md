Шаг 1. Запись уникальных данных в каждую реплику
Выполнено подключение к каждому Pod'у кластера Redis. В каждую реплику записан уникальный ключ `server`, соответствующий имени Pod'а.
Вывод консоли:
```text
$ kubectl exec -it redis-cluster-0 -n techblog -- redis-cli set server redis-0
OK
$ kubectl exec -it redis-cluster-1 -n techblog -- redis-cli set server redis-1
OK
$ kubectl exec -it redis-cluster-2 -n techblog -- redis-cli set server redis-2
OK
```
Шаг 2. Удаление Pod'а
Удалена одна из реплик кластера (`redis-cluster-1`) для проверки механизма восстановления StatefulSet.
Вывод консоли:
```text
$ kubectl delete pod redis-cluster-1 -n techblog
pod "redis-cluster-1" deleted from techblog namespace
```
Шаг 3. Анализ восстановления и проверка данных
Контроллер StatefulSet автоматически обнаружил потерю реплики и поднял новый Pod.
В отличие от Deployment, новый Pod сохранил свой строгий порядковый индекс: он снова получил имя `redis-cluster-1`.
```text
$ kubectl get pods -n techblog
NAME                        READY   STATUS    RESTARTS   AGE
blog-app-95fd8c599-qwg82    1/1     Running   0          73m
database-58c7f5fcf5-pt45c   1/1     Running   0          50m
redis-cluster-0             1/1     Running   0          3m40s
redis-cluster-1             1/1     Running   0          2s
redis-cluster-2             1/1     Running   0          3m36s
```
Выполнено чтение данных из пересозданного Pod'а:
```text
$ kubectl exec -it redis-cluster-1 -n techblog -- redis-cli get server
"redis-1"
```
