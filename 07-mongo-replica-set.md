## Предварительая конфигурация

* Развернут nfs client provisioner

## Развертывание mongodb в виде stateful set

Создайте YAML-манифест ```mongodb-statefulset.yaml```. 

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  selector:
    matchLabels:
      app: mongo
  serviceName: "mongo"
  replicas: 1
  template:
    metadata:
      labels:
        app: mongo
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: mongo
        image: mongo
        command:
        - mongod
        - "--bind_ip_all"
        - "--replSet"
        - rs0
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db
        - name: mongo-config
          mountPath: /data/configdb
  volumeClaimTemplates:
  - metadata:
      name: mongo-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
  - metadata:
      name: mongo-config
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Mi
```

Создайте YAML-манифест ```headless-service.yaml```. Данный сервис будет использоваться для формирования replication set между экземплярами базы данных и строки подключения к кластеру.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    app: mongo
spec:
  ports:
  - name: mongo
    port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    app: mongo
```

Примените созданные YAML-манифесты и проверьте, что все ресурсы активны.

```bash
kubectl create -f mongodb-statefulset.yaml -f headless-service.yaml

$ kubectl get all
NAME                                          READY   STATUS    RESTARTS         AGE
pod/mongo-0                                   1/1     Running   0                9m6s
pod/nfs-client-provisioner-67665cd7b5-bsvqj   1/1     Running   10 (3h57m ago)   3d23h

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                        AGE
service/kubelet      ClusterIP   None         <none>        10250/TCP,10255/TCP,4194/TCP   3d2h
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                        7d3h
service/mongo        ClusterIP   None         <none>        27017/TCP                      83m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-client-provisioner   1/1     1            1           3d23h

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-client-provisioner-67665cd7b5   1         1         1       3d23h

NAME                     READY   AGE
statefulset.apps/mongo   1/1     9m6s
```

## Настройка replication set в mongodb

Подключитесь к созданному контейнеру mongo-0 через mongosh.

```bash
kubectl exec -it mongo-0 -- mongosh
```

Выполните команду для инициализации replication set.

```bash
test> rs.initiate()
```

Далее, необходимо перенастроить адрес главного хоста в группе репликации. По умолчанию используется текущее имя хоста (mongo-0), но для правильной маршрутизации в кластере kubernetes необходимо использовать созданный headless service ```monogo```.

```bash
var cfg = rs.conf()
cfg.members[0].host = "mongo-0.mongo:27017"
rs.reconfig(cfg)
```

Проверьте статус группы репликации с помощью команды ```rs.status()```. Обратите внимание на ключ ```stateStr: PRIMARY```, говорящий о том, что данный экземпляр базы является главным, а также на ключ ```infoMessage``` с сообщением ```Could not find member to sync from```, говорящий о том, что в группе отсутствуют другие участники для синхронизации.

## Создание тестовой базы данных

Выполните следующие команды, чтобы создать тестовую базу, коллекцию и объект.

```bash
use pokeworld
db.createCollection("pokemons")
db.pokemons.insertOne({
...     name: "Pikachu",
...     type: "Electric",
...     health: 35
... })

pokeworld> db.pokemons.findOne({ name: "Pikachu" })
{
  _id: ObjectId("64cfbde19a94097ec579b0ca"),
  name: 'Pikachu',
  type: 'Electric',
  health: 35
}
```

## Добавление экземпляра базы данных в группу репликации

С помощью команды ```exit``` выйдите из ```mongosh``` и измените размер stateful set. Проверьте, что экземпляр ```mongo-1``` создан и активен.

```bash
kubectl scale sts mongo --replicas 2

$ kubectl get all
NAME                                          READY   STATUS    RESTARTS        AGE
pod/mongo-0                                   1/1     Running   0               21m
pod/mongo-1                                   1/1     Running   0               18m
pod/nfs-client-provisioner-67665cd7b5-bsvqj   1/1     Running   10 (4h9m ago)   4d

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                        AGE
service/kubelet      ClusterIP   None         <none>        10250/TCP,10255/TCP,4194/TCP   3d3h
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP                        7d3h
service/mongo        ClusterIP   None         <none>        27017/TCP                      96m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-client-provisioner   1/1     1            1           4d

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-client-provisioner-67665cd7b5   1         1         1       4d

NAME                     READY   AGE
statefulset.apps/mongo   2/2     21m
```

Подключитесь к созданному контейнеру mongo-0 через mongosh.

```bash
kubectl exec -it mongo-0 -- mongosh
```

Выполните команду для добавления экземпляра в группу репликации.

```bash
kubectl exec -it mongo-0 -- mongosh

rs.add("mongo-1.mongo:27017")
```

Проверьте статус группы репликации с помощью команды ```rs.status()```. В коллекции ```members``` у объекта с ```_id: 1``` обратите внимание на ключ ```stateStr: SECONDARY```, говорящий о том, что данный экземпляр базы является ведомым. 

## Проверка репликации между экземплярами

Отключитесь от ```mongo-0``` с помощью команды ```exit``` и подключитсь к ```mongo-1```. Убедитесь в том, что отсутствует лаг репликации между экземплярами.

```bash
kubectl exec -it mongo-1 -- mongosh

rs.printSecondaryReplicationInfo()
source: mongo-1.mongo:27017
{
  syncedTo: 'Sun Aug 06 2023 15:37:17 GMT+0000 (Coordinated Universal Time)',
  replLag: '0 secs (0 hrs) behind the primary '
}
```

Выберите созданную ранее на ```mongo-0``` базу и ппросмотрите список коллекций.

```bash
test> use pokeworld
pokeworld> show collections
pokemons
```

Далее, для всех приложений, использующих базу mongodb можно осуществлять подключение через строку подключения, где указаны все экземпляры группы репликации.

```bash
mongodb://mongo-0.mongo,mongo-1.mongo
```

## Очистка тестового окружения

```bash
kubectl delete -f mongodb-statefulset.yaml -f headless-service.yaml
kubectl delete pvc --all
```
