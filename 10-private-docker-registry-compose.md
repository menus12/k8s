## Предварительая конфигурация

* Развернут балансировщик нагрузки
* Развернут nginx ingress
* Развернут docker compose

## Установка локального репозитория с помощью Docker Compose

Создайте YAML-манифест ```docker-compose.yaml```

```yaml
version: '3.0'

services:

  registry:
    container_name: docker-registry
    restart: always
    image: registry:2
    ports:
      - 5000:5000
    volumes:
      - docker-registry-data:/var/lib/registry

volumes:
  docker-registry-data: {}
```

Запустите локальный репозиторий.

```bash
sudo docker compose up -d
```

Установите параметр в сервисе docker, для того чтобы можно было использовать небезопасные репозитории.

```bash
echo { "insecure-registries": ["localhost:5000"] } > /etc/docker/daemon.json
sudo systemctl restart docker
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

Выполните сборку образа и поместите образ в локальный репозиторий.

```bash
sudo docker build -t localhost:5000/mywebserver:v1 .
sudo docker push localhost:5000/mywebserver:v1
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
echo { "insecure-registries": ["kub01:5000"] } > /etc/docker/daemon.json
sudo systemctl restart docker
```

* Если рабочие узлы используют ```containerd``` в качестве container engine, необходимо отредактировать файл ```/etc/containerd/config.toml``` и добавить в него следующие директивы.

```toml
[plugins]
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
        [plugins."io.containerd.grpc.v1.cri".registry.configs."kub01:5000".tls]
          insecure_skip_verify = true
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."kub01:5000"]
          endpoint = ["http://kub01:5000"]
```

После этого необходимо перезагрузить сервис containerd.

```bash
sudo systemctl restart containerd
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

Создайте YAML-манифест ```deployment.yaml``` для описания параметров развертывания приложения и связанного с ним сервиса. Обратите внимание на ключ ```image: kub01:5000/mywebserver:v1``` в спецификации шаблона и конфигурацию переменных окружения.

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
        image: kub01:5000/mywebserver:v1
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

Создайте YAML-манифест ```ingress-resource.yaml``` для возможности проверки работы приложения из вне.

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

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
service/fastapi-service   ClusterIP   10.107.59.53    <none>        80/TCP                         42m
service/kubelet           ClusterIP   None            <none>        10250/TCP,10255/TCP,4194/TCP   5d20h
service/kubernetes        ClusterIP   10.96.0.1       <none>        443/TCP                        9d

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/fastapi                  3/3     3            3           42m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/fastapi-7975c76769                  3         3         3       42m

$ kubectl describe ing
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
kubectl delete -f deployment.yaml -f ingress-resource.yaml -f configmap.yaml -f secret.yaml
sudo docker compose down
sudo docker image rm registry:2
sudo docker image rm localhost:5000/mywebserver:v1
sudo docker image prune
```
