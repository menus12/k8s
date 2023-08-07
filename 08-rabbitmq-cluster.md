## Предварительая конфигурация

* Развернут балансировщик нагрузки
* Развернут nginx ingress

## Добавление локального класса хранения

Примените YAML-манифест для возможности автоматического развертывания локальных PVC.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml
```

Проверьте параметры развернутого класса хранения.

```bash
$ kubectl get sc
NAME                            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path                      rancher.io/local-path   Delete          WaitForFirstConsumer   false                  171m
```

## Установка rabbitmq

Создайте YAML-манифест ```rbac.yaml``` в котором необходимо определить сервисный аккаунт, роль и привящку роли к сервисному аккаунту.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rabbitmq
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rabbitmq
rules:
- apiGroups:
    - ""
  resources:
    - endpoints
  verbs:
    - get
    - list
    - watch
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rabbitmq
subjects:
- kind: ServiceAccount
  name: rabbitmq
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rabbitmq
```

Создайте YAML-манифест ```secret.yaml``` в котором необходимо указать erlang cookie для аутентификации внутри кластера rabbitmq.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbit-secret
type: Opaque
data:
  # echo -n "cookie-value" | base64
  RABBITMQ_ERLANG_COOKIE: V0lXVkhDRFRDSVVBV0FOTE1RQVc=
```

Создайте YAML-манифест ```rabbit-configmap.yaml``` в котором описываются параметры запуска экземпляров rabbitmq.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-config
data:
  enabled_plugins: |
    [rabbitmq_federation,rabbitmq_management,rabbitmq_peer_discovery_k8s].
  rabbitmq.conf: |
    loopback_users.guest = false
    listeners.tcp.default = 5672

    cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
    cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
    cluster_formation.k8s.address_type = hostname
    cluster_formation.node_cleanup.only_log_warning = true
```

Создайте YAML-манифест ```sts.yaml``` в котором описываются параметры stateful set для rabbitmq.

```bash
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
spec:
  serviceName: rabbitmq
  replicas: 2
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      serviceAccountName: rabbitmq
      initContainers:
      - name: config
        image: busybox
        command: ['/bin/sh', '-c', 'cp /tmp/config/rabbitmq.conf /config/rabbitmq.conf && ls -l /config/ && cp /tmp/config/enabled_plugins /etc/rabbitmq/enabled_plugins']
        volumeMounts:
        - name: config
          mountPath: /tmp/config/
          readOnly: false
        - name: config-file
          mountPath: /config/
        - name: plugins-file
          mountPath: /etc/rabbitmq/
      containers:
      - name: rabbitmq
        image: rabbitmq:3.11-management
        ports:
        - containerPort: 4369
          name: discovery
        - containerPort: 5672
          name: amqp
        env:
        - name: RABBIT_POD_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        - name: RABBIT_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: RABBITMQ_NODENAME
          value: rabbit@$(RABBIT_POD_NAME).rabbitmq.$(RABBIT_POD_NAMESPACE).svc.cluster.local
        - name: RABBITMQ_USE_LONGNAME
          value: "true"
        - name: RABBITMQ_CONFIG_FILE
          value: "/config/rabbitmq"
        - name: RABBITMQ_ERLANG_COOKIE
          valueFrom:
            secretKeyRef:
              name: rabbit-secret
              key: RABBITMQ_ERLANG_COOKIE
        - name: K8S_HOSTNAME_SUFFIX
          value: .rabbitmq.$(RABBIT_POD_NAMESPACE).svc.cluster.local
        volumeMounts:
        - name: data
          mountPath: /var/lib/rabbitmq
          readOnly: false
        - name: config-file
          mountPath: /config/
        - name: plugins-file
          mountPath: /etc/rabbitmq/
      volumes:
      - name: config-file
        emptyDir: {}
      - name: plugins-file
        emptyDir: {}
      - name: config
        configMap:
          name: rabbitmq-config
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "local-path"
      resources:
        requests:
          storage: 50Mi
---
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  clusterIP: None
  ports:
  - port: 4369
    targetPort: 4369
    name: discovery
  - port: 5672
    targetPort: 5672
    name: amqp
  - port: 15672
    targetPort: 15672
    name: http
  selector:
    app: rabbitmq
```

Создайте YAML-манифест ```ingress-resource.yaml``` для возможности доступа к веб-интерфейсу кластера из внешней сети.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-nginx
spec:
  ingressClassName: nginx
  rules:
  - host: rabbitmq.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rabbitmq
            port:
              number: 15672
```

Примените созданные YAML-манифесты и проверьте, что все ресурсы созданны и активны.

```bash
kubectl create -f ingress-resource.yaml -f rbac.yaml -f secret.yaml -f rabbit-configmap.yaml -f sts.yaml  -f ingress-resource.yaml

$ kubectl get all
NAME                                          READY   STATUS    RESTARTS         AGE
pod/rabbitmq-0                                1/1     Running   0                25m
pod/rabbitmq-1                                1/1     Running   0                55m

NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                        AGE
service/kubelet      ClusterIP      None             <none>            10250/TCP,10255/TCP,4194/TCP   3d19h
service/kubernetes   ClusterIP      10.96.0.1        <none>            443/TCP                        7d19h
service/rabbitmq     ClusterIP      None             <none>            4369/TCP,5672/TCP,15672/TCP    55m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-client-provisioner   1/1     1            1           4d16h

NAME                        READY   AGE
statefulset.apps/rabbitmq   2/2     55m
```

## Настройка политики репликации в кластере rabbitmq

Войдите в терминал экземпляра ```rabbitmq-0``` и выполните команду, для создания политики репликации. Обратите внимание на полные доменные имена сервисов — если экземпляры располагаются не в пространстве имен по умолчанию, следует подставить пространство имен вместо ```default```.

```bash
kubectl exec -it rabbitmq-0 -- bash

rabbitmqctl set_policy ha-fed \
    ".*" '{"federation-upstream-set":"all", "ha-sync-mode":"automatic", "ha-mode":"nodes", "ha-params":["rabbit@rabbitmq-0.rabbitmq.default.svc.cluster.local","rabbit@rabbitmq-1.rabbitmq.default.svc.cluster.local"]}' \
    --priority 1 \
    --apply-to queues
```

## Проверка работы кластера rabbitmq

Проверьте, какой IP-адрес используется ingress-контроллером для балансировки нагрузки.

```bash
 kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.178.253   192.168.133.160   80:30562/TCP,443:31608/TCP   44h
ingress-nginx-controller-admission   ClusterIP      10.96.120.252    <none>            443/TCP                      44h
```

Добавьте тестовые DNS-записи в файл hosts.

```bash
192.168.133.160 rabbitmq.example.com
```

Создайте контейнер с утилитой curl внутри кластера, для тестирования публикации и получения сообщений в кластере. Параллельно можно открыть веб-браузер по адресу ```http://rabbitmq.example.com```, войти под учетной записью ```guest:guest``` и наблюдать за изменениями в кластере.

```bash
kubectl run mycurlpod --image=curlimages/curl -i --tty -- sh
```

Сделайте запросы к экземпляру ```rabbitmq-0``` — создайте точку обмена (exchange) ```testex```, очередь ```testq``` и привяжите точку обмена к очереди.

```bash
curl -i -u guest:guest -H "content-type:application/json" -X PUT http://rabbitmq-0.rabbitmq:15672/api/exchanges/%2f/testex -d'{"type":"direct","auto_delete":false,"durable":true,"internal":false,"arguments":{}}'

curl -i -u guest:guest -H "content-type:application/json" -X PUT http://rabbitmq-0.rabbitmq:15672/api/queues/%2f/testq -d'{"auto_delete":false,"durable":true,"arguments":{}}'

curl -i -u guest:guest -H "content-type:application/json" -X POST http://rabbitmq-0.rabbitmq:15672/api/bindings/%2f/e/testex/q/testq -d '{"routing_key":"","arguments”":{}}'
```

Далее, сделайте запрос к экземпляру ```rabbitmq-0```, чтобы опубликовать сообщение ```Hello to rabbitmq-0``` в созданной очереди. В случае успешной публикации вернется ответ ```{"routed":true}```.

```bash
curl -s -u guest:guest -H "Accept: application/json" -H "Content-Type:application/json" -X POST -d'{
    "vhost": "/",
    "name": "testex",
    "properties": {
        "delivery_mode": 2,
        "headers": {}
    },
    "routing_key": "",
    "delivery_mode": "1",
    "payload":"Hello to rabbitmq-0",
    "headers": {},
    "props": {},
    "payload_encoding": "string"
}' http://rabbitmq-0.rabbitmq:15672/api/exchanges/%2F/testex/publish
```

Далее, сделайте запрос к экземпляру ```rabbitmq-1```, чтобы получить сообщение из созданной очереди.

```bash
$ curl -u guest:guest -i -H "content-type:application/json" -X POST http://rabbitmq-1.rabbitmq:15672/api/queues/%2F/testq/get -d'{"count
":1,"ackmode":"ack_requeue_false","encoding":"auto","truncate":50000}'
HTTP/1.1 200 OK
cache-control: no-cache
content-length: 203
content-security-policy: script-src 'self' 'unsafe-eval' 'unsafe-inline'; object-src 'self'
content-type: application/json
date: Mon, 07 Aug 2023 08:46:35 GMT
server: Cowboy
vary: accept, accept-encoding, origin

[{"payload_bytes":19,"redelivered":false,"exchange":"testex","routing_key":"","message_count":0,"properties":{"delivery_mode":2,"headers":{}},"payload":"Hello to rabbitmq-0","payload_encoding":"string"}]
```

Далее, сделайте запрос к экземпляру ```rabbitmq-1```, чтобы опубликовать сообщение ```Hello to rabbitmq-1``` в созданной очереди. В случае успешной публикации вернется ответ ```{"routed":true}```.

```bash
curl -s -u guest:guest -H "Accept: application/json" -H "Content-Type:application/json" -X POST -d'{
    "vhost": "/",
    "name": "testex",
    "properties": {
        "delivery_mode": 2,
        "headers": {}
    },
    "routing_key": "",
    "delivery_mode": "1",
    "payload":"Hello to rabbitmq-1",
    "headers": {},
    "props": {},
    "payload_encoding": "string"
}' http://rabbitmq-1.rabbitmq:15672/api/exchanges/%2F/testex/publish
```

Далее, сделайте запрос к экземпляру ```rabbitmq-0```, чтобы получить сообщение из созданной очереди.

```bash
$ curl -u guest:guest -i -H "content-type:application/json" -X POST http://rabbitmq-0.rabbitmq:15672/api/queues/%2F/testq/get -d'{"count
":1,"ackmode":"ack_requeue_false","encoding":"auto","truncate":50000}'
HTTP/1.1 200 OK
cache-control: no-cache
content-length: 203
content-security-policy: script-src 'self' 'unsafe-eval' 'unsafe-inline'; object-src 'self'
content-type: application/json
date: Mon, 07 Aug 2023 08:48:34 GMT
server: Cowboy
vary: accept, accept-encoding, origin

[{"payload_bytes":19,"redelivered":false,"exchange":"testex","routing_key":"","message_count":0,"properties":{"delivery_mode":2,"headers":{}},"payload":"Hello to rabbitmq-1","payload_encoding":"string"}]
```

Далее, сделайте запрос к headless-сервису ```rabbitmq```, чтобы опубликовать сообщение ```Hello to rabbitmq-any``` в созданной очереди. В случае успешной публикации вернется ответ ```{"routed":true}```.

```bash
curl -s -u guest:guest -H "Accept: application/json" -H "Content-Type:application/json" -X POST -d'{
    "vhost": "/",
    "name": "testex",
    "properties": {
        "delivery_mode": 2,
        "headers": {}
    },
    "routing_key": "",
    "delivery_mode": "1",
    "payload":"Hello to rabbitmq-any",
    "headers": {},
    "props": {},
    "payload_encoding": "string"
}' http://rabbitmq:15672/api/exchanges/%2F/testex/publish
```

Далее, сделайте запрос к headless-сервису ```rabbitmq```, чтобы получить сообщение из созданной очереди.

```bash
$ curl -u guest:guest -i -H "content-type:application/json" -X POST http://rabbitmq:15672/api/queues/%2F/testq/get -d'{"count":1,"ackmod
e":"ack_requeue_false","encoding":"auto","truncate":50000}'
HTTP/1.1 200 OK
cache-control: no-cache
content-length: 205
content-security-policy: script-src 'self' 'unsafe-eval' 'unsafe-inline'; object-src 'self'
content-type: application/json
date: Mon, 07 Aug 2023 08:50:53 GMT
server: Cowboy
vary: accept, accept-encoding, origin

[{"payload_bytes":21,"redelivered":false,"exchange":"testex","routing_key":"","message_count":0,"properties":{"delivery_mode":2,"headers":{}},"payload":"Hello to rabbitmq-any","payload_encoding":"string"}]
```

Далее, еще раз сделайте запрос к экземпляру ```rabbitmq-0```, чтобы опубликовать сообщение ```Hello to rabbitmq-0``` в созданной очереди. Откройте еще один терминал и подключитесь к экземпляру ```rabbitmq-0``` и проверьте, что созданная очередь зеркалируется вторым экземпляром.

```bash
kubectl exec -it rabbitmq-0 -- bash

# rabbitmqctl list_queues name policy pid mirror_pids
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name    policy  pid     mirror_pids
testq   ha-fed  <rabbit@rabbitmq-0.rabbitmq.default.svc.cluster.local.1691393525.1481.0>        [<rabbit@rabbitmq-1.rabbitmq.default.svc.cluster.local.1691395296.550.0>]
```

Выйдите из консоли ```rabitmq-0``` и удалите pod. В это время в консоли контейнера с curl попробуйте получить последнее сообщение, обративших к headless-сервису ```rabbitmq```. Сообщение должно быть получено успешно. После пересоздания pod'а ```rabbitmq-0``` снова войдите в консоль и проверьте, что основным экземпляром для созданной очереди выступает ```rabbitmq-1```, а ```rabbitmq-0``` является зеркалирующим экземпляром.

## Очистка тестового окружения

```bash
kubectl delete -f ingress-resource.yaml -f rbac.yaml -f secret.yaml -f rabbit-configmap.yaml -f sts.yaml  -f ingress-resource.yaml
kubectl delete pvc --all
kubectl delete -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml
```
