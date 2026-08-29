
## 1. Вывод команд kubectl get pv, pvc, storageclass -n techblog
```
$ kubectl get pv,pvc -n techblog
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                    STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
persistentvolume/blog-data-pv                               5Gi        RWO            Retain           Bound         techblog/blog-data-pvc   manual         <unset>                          57m
persistentvolume/cache-pv                                   2Gi        RWO            Delete           Available                              fast-storage   <unset>                          3m47s
persistentvolume/data-pv-01                                 5Gi        RWO            Retain           Available                                             <unset>                          6d9h
persistentvolume/pvc-264fafa9-049a-424a-9ae1-4a112ce964ae   5Gi        RWO            Delete           Terminating   default/data-mysql-4     standard       <unset>                          24h
persistentvolume/pvc-2ec72f47-b04d-4807-8aa2-0fdb3d2117e0   5Gi        RWO            Delete           Terminating   default/data-mysql-1     standard       <unset>                          24h
persistentvolume/pvc-51e9798c-cbc9-474d-b889-b988bde8b7af   5Gi        RWO            Delete           Terminating   default/data-mysql-2     standard       <unset>                          24h
persistentvolume/pvc-736f56ff-14f4-4575-a5ac-061e3a703b9e   8Gi        RWO            Delete           Terminating   default/postgres-pvc     standard       <unset>                          6d7h
persistentvolume/pvc-aad7f2e9-eb4c-4da1-82b3-544f63fcf526   5Gi        RWO            Delete           Terminating   default/data-mysql-3     standard       <unset>                          24h
persistentvolume/pvc-f0c2aca5-e221-4f49-aaa5-fdd7ead7ccfb   5Gi        RWO            Delete           Terminating   default/data-mysql-0     standard       <unset>                          24h
persistentvolume/uploads-pv                                 10Gi       RWX            Retain           Bound         techblog/uploads-pvc     manual         <unset>                          51m

NAME                                      STATUS    VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/blog-data-pvc       Bound     blog-data-pv   5Gi        RWO            manual         <unset>                 55m
persistentvolumeclaim/cache-storage-pvc   Pending                                            fast-storage   <unset>                 3m41s
persistentvolumeclaim/uploads-pvc         Bound     uploads-pv     10Gi       RWX            manual         <unset>                 49m
```

## 2. Результаты тестирования персистентности данных в PostgreSQL
Тестирование пройдено успешно. Данные полностью сохраняются при удалении Pod'а базы данных, так как жизненный цикл данных (PersistentVolume) отделен от жизненного цикла Pod'ов (Deployment)

## 3. Сравнение поведения различных типов volumes (emptyDir, hostPath, PVC)

* **emptyDir (`/tmp/cache`)**: Диск создается вместе с Pod'ом и удаляется при смерти Pod'а. Данные не переживают рестарт контейнера (подходит только для временного кэша)
* **hostPath (`/var/log/app`)**: Данные пишутся напрямую на жесткий диск ноды, на которой запущен Pod. Переживают рестарт Pod'а, но если Кубер поднимет новый Pod на другой ноде, то данные будут недоступны
* **PersistentVolumeClaim (`blog-data-pvc`)**: Переживает смерть Pod'ов и перезапуски на других нодах. Кубер сам заботится о монтировании нужного диска к нужному Pod'у

## 4. Демонстрация уникальности PVC для каждого Pod'а в StatefulSet
Благодаря механизму `volumeClaimTemplates`, Кубер автоматически создал уникальные заявки (PVC) для каждой реплики StatefulSet. Имена формируются по строгому паттерну `<имя-шаблона>-<имя-пода>`.

## 5. Объяснение различий между Deployment и StatefulSet в контексте хранилища
1. **Разделение дисков (Sharing):** 
   * **Deployment:** Если мы подключим PVC к Deployment и сделаем 3 реплики, все 3 Pod'а попытаются примонтировать один и тот же физический диск 
   * **StatefulSet:** Гарантирует, что каждая реплика получит свой собственный независимый диск через механизм шаблонов (`volumeClaimTemplates`)

2. **Жизненный цикл и Именования:**
   * **Deployment:** Pod'ы эфемерны, получают случайные хэши в имени. При пересоздании нет гарантии, что Pod поднимется на той же ноде и с тем же именем
   * **StatefulSet:** Pod'ы имеют строгий порядковый номер  и поднимаются по очереди. При удалении Pod'а, новый Pod поднимется с тем же именем и автоматически подцепит тот же самый диск, сохранив состояние приложения

## 6. Результаты тестирования масштабирования StatefulSet
При выполнении `kubectl scale statefulset redis-cluster --replicas=2` был корректно удален Pod с наибольшим индексом (`redis-cluster-2`)
Однако связанный с ним PersistentVolumeClaim (`redis-data-redis-cluster-2`) остался в кластере. Это встроенная защита StatefulSet: масштабирование вниз не приводит к удалению данных
При обратном масштабировании до 3 реплик, новый Pod `redis-cluster-2` автоматически подхватил оставленный PVC, и все записанные ранее данные (`"redis-2"`) оказались на месте.

