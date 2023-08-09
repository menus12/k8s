## Предварительая конфигурация

* Развернут балансировщик нагрузки
* Развернут nginx ingress
* Развернут любой storage class по умолчанию

## Установка локального репозитория в kubernetes-кластер

Создайте YAML-манифест ```deployment.yaml```, где будут описаны pvc, deployment и соответствующий сервис для локального репозитория.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    io.kompose.service: docker-registry-data
  name: docker-registry-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    io.kompose.service: registry
  name: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      io.kompose.service: registry
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        io.kompose.network/registry-default: "true"
        io.kompose.service: registry
    spec:
      containers:
        - image: registry:2
          name: docker-registry
          ports:
            - containerPort: 5000
              hostPort: 5000
              protocol: TCP
          volumeMounts:
            - mountPath: /var/lib/registry
              name: docker-registry-data
      restartPolicy: Always
      volumes:
        - name: docker-registry-data
          persistentVolumeClaim:
            claimName: docker-registry-data
---
apiVersion: v1
kind: Service
metadata:
  labels:
    io.kompose.service: registry
  name: registry
spec:
  ports:
    - name: "5000"
      port: 5000
      targetPort: 5000
  selector:
    io.kompose.service: registry
```

Создайте YAML-манифест ```ingress-resource.yaml``` для публикации локального репозитория.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  rules:
  - host: registry.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: registry
            port:
              number: 5000
```

Примените созданные манифесты и проверьте, что ресурсы созданы и активны.

```bash
kubectl create -f deployment.yaml -f ingress-resource.yaml

$ kubectl get all
NAME                                          READY   STATUS    RESTARTS        AGE
pod/registry-6b8b57688b-9rlxw                 1/1     Running   0               82m

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
service/kubelet           ClusterIP   None            <none>        10250/TCP,10255/TCP,4194/TCP   5d21h
service/kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP                        9d
service/registry          ClusterIP   10.110.32.120   <none>        5000/TCP                       82m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/registry                 1/1     1            1           82m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/registry-6b8b57688b                 1         1         1       82m

$ kubectl get pvc docker-registry-data
NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
docker-registry-data   Bound    pvc-28b27f22-d211-4e25-8d06-43f418c68182   2Gi        RWO            managed-nfs-storage   83m

$ kubectl describe ing ingress-resource
Name:             ingress-resource
Labels:           <none>
Namespace:        default
Address:          192.168.133.160
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host                  Path  Backends
  ----                  ----  --------
  registry.example.com
                        /   registry:5000 (10.244.1.229:5000)
Annotations:            nginx.ingress.kubernetes.io/proxy-body-size: 0
                        nginx.ingress.kubernetes.io/proxy-read-timeout: 600
                        nginx.ingress.kubernetes.io/proxy-send-timeout: 600
Events:                 <none>
```

## Создание образа контейнера

Создайте файл приложения ```main.py``` для Python FastAPI.

```python
from fastapi import FastAPI
import os

app = FastAPI()

hostname = os.environ.get('HOSTNAME', 'not set')
custom_env = os.environ.get('CUSTOM_ENV', 'not set')
custom_secret = os.environ.get('CUSTOM_SECRET', 'not set')


@app.get("/")
async def root():
    return {"host": hostname, "custom_env": custom_env, "custom_secret": custom_secret}
```

Создайте ```Dockerfile``` для создания локального образа контейнера.

```dockerfile
FROM python:3.9
WORKDIR /code
RUN pip install --no-cache-dir --upgrade fastapi
RUN pip install uvicorn "uvicorn[standard]"
COPY ./main.py /code/main.py
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "80"]
```

Установите параметр в сервисе docker, для того чтобы можно было использовать небезопасные репозитории.

```bash
echo { "insecure-registries": ["registry.example.com:80"] } > /etc/docker/daemon.json
sudo systemctl restart docker
```

Добавьте DNS-записи в локальный файл hosts.

```bash
192.168.133.160 registry.example.com
```

Выполните сборку образа и поместите образ в локальный репозиторий.

```bash
sudo docker build -t registry.example.com:80/mywebserver:v1 .
sudo docker push registry.example.com:80/mywebserver:v1
```

Запустите контейнер и проверьте его работу, затем остановите и удалите контейнер.

```bash
sudo docker run --name web -d localhost:5000/mywebserver:v1

$ sudo docker inspect web | grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.2",
                    "IPAddress": "172.17.0.2",

$ curl 172.17.0.2
{"host":"ce861a91125e","custom_env":"not set","custom_secret":"not set"}

sudo docker stop web
sudo docker remove web
```

## Конфигурация рабочих узлов

Для запуска контейнеров в kubernetes необходимо разрешить использование небезопасных репозиториев. В данном примере локальный репозиторий расположен на ```kub01```.

* Если рабочие узлы используют ```docker``` в качестве container engine:

```bash
echo { "insecure-registries": ["registry.example.com:80"] } > /etc/docker/daemon.json
sudo systemctl restart docker
```

* Если рабочие узлы используют ```containerd``` в качестве container engine, необходимо отредактировать файл ```/etc/containerd/config.toml``` и добавить в него следующие директивы.

```toml
[plugins]
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."registry.example.com:80".tls]
          insecure_skip_verify = true
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.example.com:80"]
          endpoint = ["http://registry.example.com:80"]
```

После этого необходимо перезагрузить сервис containerd.

```bash
sudo systemctl restart containerd
```

Добавьте DNS-записи в локальный файл hosts на всех рабочих узлах.

```bash
192.168.133.160 registry.example.com
```

## Развертывание приложения из локального репозитория

Создайте YAML-манифест ```configmap.yaml``` для конфигурации окружения.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mywebserver-config
data:
  greeting: hello world
```

Создайте YAML-манифест ```secret.yaml``` для конфигурации секретов для приложения.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mywebserver-secret
type: Opaque
stringData:
  password: "P@ssw0rd"
```

Примените созданные манифесты и проверьте, что они активны.

```bash
kubectl create -f configmap.yaml -f secret.yaml

$ kubectl describe cm mywebserver-config
Name:         mywebserver-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
greeting:
----
hello world

BinaryData
====

Events:  <none>

$ kubectl describe secret mywebserver-secret
Name:         mywebserver-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  8 bytes
```

Создайте YAML-манифест ```deployment-fastapi.yaml``` для описания параметров развертывания приложения и связанного с ним сервиса. Обратите внимание на ключ ```image: kub01:5000/mywebserver:v1``` в спецификации шаблона и конфигурацию переменных окружения.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi
  labels:
    app: fastapi
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi
  template:
    metadata:
      labels:
        app: fastapi
    spec:
      containers:
      - name: fastapi
        image: registry.example.com:80/mywebserver:v1
        imagePullPolicy: Always
        env:
          - name: CUSTOM_ENV
            valueFrom:
              configMapKeyRef:
                name: mywebserver-config
                key: greeting
          - name: CUSTOM_SECRET
            valueFrom:
              secretKeyRef:
                name: mywebserver-secret
                key: password
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: fastapi-service
spec:
  selector:
    app: fastapi
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Создайте YAML-манифест ```ingress-fastapi.yaml``` для возможности проверки работы приложения из вне.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-fastapi
spec:
  rules:
  - host: fastapi.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fastapi-service
            port:
              number: 80
```

Примените созданные YAML-манифесты и проверьте, что все ресурсы созданы и активны. Для отладки развертывания в соседнем терминале выполните команду ```kubectl get events -w```.

```bash
kubectl create -f deployment.yaml -f ingress-resource.yaml

$ kubectl get all
NAME                                          READY   STATUS    RESTARTS        AGE
pod/fastapi-7975c76769-2kgjg                  1/1     Running   0               42m
pod/fastapi-7975c76769-rwb9r                  1/1     Running   0               42m
pod/fastapi-7975c76769-tqpqm                  1/1     Running   0               42m
pod/registry-6b8b57688b-9rlxw                 1/1     Running   0               61m

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
service/fastapi-service   ClusterIP   10.107.59.53    <none>        80/TCP                         42m
service/kubelet           ClusterIP   None            <none>        10250/TCP,10255/TCP,4194/TCP   5d20h
service/kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP                        9d

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/fastapi                  3/3     3            3           42m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/fastapi-7975c76769                  3         3         3       42m

$ kubectl describe ing ingress-fastapi
Name:             ingress-fastapi
Labels:           <none>
Namespace:        default
Address:          192.168.133.160
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host                 Path  Backends
  ----                 ----  --------
  fastapi.example.com
                       /   fastapi-service:80 (10.244.1.236:80,10.244.1.237:80,10.244.1.238:80)
Annotations:           <none>
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    41m (x2 over 41m)  nginx-ingress-controller  Scheduled for sync
```

## Проверка работы приложения

Внесите DNS-запись в локальный файл hosts.

```bash
192.168.133.160   fastapi.example.com
```

Сделайте 3 запроса по адресу ```fastapi.example.com```, обратите внимание, на то, что приложение корректно использует переменные среды из ```configMap``` и ```secrets`.

```bash
$ curl fastapi.example.com
{"host":"fastapi-7975c76769-rwb9r","custom_env":"hello world","custom_secret":"P@ssw0rd"}

$ curl fastapi.example.com
{"host":"fastapi-7975c76769-tqpqm","custom_env":"hello world","custom_secret":"P@ssw0rd"}

$ curl fastapi.example.com
{"host":"fastapi-7975c76769-2kgjg","custom_env":"hello world","custom_secret":"P@ssw0rd"}
```

## Очистка тестового окружения

```bash
kubectl delete -f deployment.yaml -f ingress-resource.yaml -f deployment-fastapi.yaml -f ingress-fastapi.yaml -f configmap.yaml -f secret.yaml
sudo docker image rm registry.example.com:80/mywebserver:v1
sudo docker image prune
```
