
Шаг 1. Подключение к БД и генерация тестовых данных
Выполнен вход в интерактивный терминал запущенного Pod'а `database-58c7f5fcf5-wdmtg`. Создана таблица `test_data` и внесена тестовая запись.
Вывод консоли:
```text
$ kubectl exec -it database-58c7f5fcf5-wdmtg -n techblog -- psql -U bloguser -d techblog

techblog=# CREATE TABLE test_data (id serial PRIMARY KEY, text VARCHAR(50));
CREATE TABLE

techblog=# INSERT INTO test_data (text) VALUES ('Данные выжили после удаления пода!');
INSERT 0 1

techblog=# SELECT * FROM test_data;
 id |                text
----+------------------------------------
  1 | Данные выжили после удаления пода!
```
Шаг 2. Имитация сбоя (Удаление Pod'а)
Текущий Pod базы данных был принудительно удален для проверки работы ReplicaSet и переподключения Volumes.
Вывод консоли:
```text
$ kubectl delete pod database-58c7f5fcf5-wdmtg -n techblog
pod "database-58c7f5fcf5-wdmtg" deleted

$ kubectl get pods -n techblog
NAME                        READY   STATUS    RESTARTS   AGE
blog-app-95fd8c599-qwg82    1/1     Running   0          23m
database-58c7f5fcf5-pt45c   1/1     Running   0          9s
```
Шаг 3. Проверка сохранности данных
Выполнен запрос на чтение таблицы `test_data` из нового Pod'а `database-58c7f5fcf5-pt45c`.
Вывод консоли:
```text
$ kubectl exec -it database-58c7f5fcf5-pt45c -n techblog -- psql -U bloguser -d techblog -c "SELECT * FROM test_data;"
 id |                text
----+------------------------------------
  1 | Данные выжили после удаления пода!
(1 row)
```
