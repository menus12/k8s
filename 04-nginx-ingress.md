## Предварительая конфигурация

* Развернут балансировщик нагрузки
* Установлен Helm v3

## Установка Nginx Ingress Controller

С помощью Helm сохраните файл со значениями для возможности его локального редактирования.

```bash
helm show values ingress-nginx --repo https://kubernetes.github.io/ingress-nginx > ingress-nginx-values.yaml
```

Отредактируйте файл ```ingress-nginx-values.yaml```. В директиве ```ingressClassResource``` измените значения ключа ```default``` с ```false``` на ```true```. 

```yaml
ingressClassResource:
   # -- Name of the ingressClass
   name: nginx
   # -- Is this ingressClass enabled or not
   enabled: true
   # -- Is this the default ingressClass for the cluster
   default: true
   # -- Controller-value of the controller that is processing this ingressClass
   controllerValue: "k8s.io/ingress-nginx"
   # -- Parameters is a link to a custom resource containing additional
   # configuration for the controller. This is optional if the controller
   # does not require extra parameters.
   parameters: {}
```

Установите ```ingress-nginx``` с помощью Helm, используя локальный файл со значениями ```ingress-nginx-values.yaml```. Все компоненты будут установлены в пространстве имен ```ingress-nginx```.

```bash
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace --values ingress-nginx-values.yaml
```

Проверьте, что все компоненты установлены и активны.

```bash
$ kubectl get all -n ingress-nginx
NAME                                            READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-5fcb5746fc-s9plw   1/1     Running   0          6m44s

NAME                                         TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.110.174.232   192.168.133.160   80:32338/TCP,443:32433/TCP   6m44s
service/ingress-nginx-controller-admission   ClusterIP      10.97.251.30     <none>            443/TCP                      6m44s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           6m44s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-5fcb5746fc   1         1         1       6m44s
```

Проверьте, что Ingress Class ```nginx``` помечен классом по умолчанию.

```bash
$ kubectl get ingressclass
NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       74s

$ kubectl describe ingressclass
Name:         nginx
Labels:       app.kubernetes.io/component=controller
              app.kubernetes.io/instance=ingress-nginx
              app.kubernetes.io/managed-by=Helm
              app.kubernetes.io/name=ingress-nginx
              app.kubernetes.io/part-of=ingress-nginx
              app.kubernetes.io/version=1.8.1
              helm.sh/chart=ingress-nginx-4.7.1
Annotations:  ingressclass.kubernetes.io/is-default-class: true
              meta.helm.sh/release-name: ingress-nginx
              meta.helm.sh/release-namespace: ingress-nginx
Controller:   k8s.io/ingress-nginx
Events:       <none>
```

## Развертывание тестовых приложений

Для проверки работы ingress-контроллера разверните 3 тестовых приложения на базе nginx.

* nginx-deploy-main.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-main
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-main
  template:
    metadata:
      labels:
        run: nginx-main
    spec:
      containers:
      - image: nginx
        name: nginx
```

* nginx-deploy-blue.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-blue
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-blue
  template:
    metadata:
      labels:
        run: nginx-blue
    spec:
      volumes:
      - name: webdata
        emptyDir: {}
      initContainers:
      - name: web-content
        image: busybox
        volumeMounts:
        - name: webdata
          mountPath: "/webdata"
        command: ["/bin/sh", "-c", 'echo "<h1>I am <font color=blue>BLUE</font></h1>" > /webdata/index.html']
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: webdata
          mountPath: "/usr/share/nginx/html"
```

* nginx-deploy-green.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy-green
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx-green
  template:
    metadata:
      labels:
        run: nginx-green
    spec:
      volumes:
      - name: webdata
        emptyDir: {}
      initContainers:
      - name: web-content
        image: busybox
        volumeMounts:
        - name: webdata
          mountPath: "/webdata"
        command: ["/bin/sh", "-c", 'echo "<h1>I am <font color=green>GREEN</font></h1>" > /webdata/index.html']
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: webdata
          mountPath: "/usr/share/nginx/html"
```

Примените созданные YAML-манифесты.

```bash
kubectl create -f nginx-deploy-main.yaml -f nginx-deploy-blue.yaml -f nginx-deploy-green.yaml
```

Опубликуйте сервис для каждого приложения на порту 80.

```bash
kubectl expose deploy nginx-deploy-main --port 80
kubectl expose deploy nginx-deploy-blue --port 80
kubectl expose deploy nginx-deploy-green --port 80
```

Проверьте, что все приложения и сервисы активны.

```bash
$ kubectl get all
NAME                                          READY   STATUS    RESTARTS      AGE
pod/nginx-deploy-blue-75cc67f786-wqq2z        1/1     Running   0             83s
pod/nginx-deploy-green-66c988648c-4wkz6       1/1     Running   0             78s
pod/nginx-deploy-main-6c594975bc-d4tzp        1/1     Running   0             4m10s

NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                        AGE
service/kubelet              ClusterIP   None             <none>        10250/TCP,10255/TCP,4194/TCP   26h
service/kubernetes           ClusterIP   10.96.0.1        <none>        443/TCP                        5d3h
service/nginx-deploy-blue    ClusterIP   10.104.130.243   <none>        80/TCP                         60s
service/nginx-deploy-green   ClusterIP   10.101.102.91    <none>        80/TCP                         54s
service/nginx-deploy-main    ClusterIP   10.110.208.122   <none>        80/TCP                         3m37s

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-deploy-blue        1/1     1            1           83s
deployment.apps/nginx-deploy-green       1/1     1            1           78s
deployment.apps/nginx-deploy-main        1/1     1            1           4m10s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-deploy-blue-75cc67f786        1         1         1       83s
replicaset.apps/nginx-deploy-green-66c988648c       1         1         1       78s
replicaset.apps/nginx-deploy-main-6c594975bc        1         1         1       4m10s
```

## Создание Ingress Resourse

Создайте YAML-манифест ```ingress-resource.yaml```. Данные правида будут использоваться для маршрутизации запросов к опубликованым сервисам по DNS-именам:

* http://nginx.example.com --> nginx-deploy-main:80
* http://blue.nginx.example.com --> nginx-deploy-blue:80
* http://green.nginx.example.com --> nginx-deploy-green:80

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-nginx
spec:
  ingressClassName: nginx
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-deploy-main
            port:
              number: 80
  - host: blue.nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-deploy-blue
            port:
              number: 80
  - host: green.nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-deploy-green
            port:
              number: 80
```

Примените созданный YAML-манифест и проверьте его параметры.

```bash
kubectl create -f ingress-resource.yaml

$ kubectl describe ing ingress-resource-nginx
Name:             ingress-resource-nginx
Labels:           <none>
Namespace:        default
Address:          192.168.133.160
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host                     Path  Backends
  ----                     ----  --------
  nginx.example.com          /   nginx-deploy-main:80 (10.244.1.67:80)
  blue.nginx.example.com     /   nginx-deploy-blue:80 (10.244.1.68:80)
  green.nginx.example.com    /   nginx-deploy-green:80 (10.244.1.69:80)
Annotations:               <none>
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    11s (x2 over 35s)  nginx-ingress-controller  Scheduled for sync
```

## Проверка работы ingress resourse

Добавьте тестовые DNS-записи в файл hosts.

```bash
192.168.133.160 nginx.example.com
192.168.133.160 blue.nginx.example.com
192.168.133.160 green.nginx.example.com
```

Сделайте http-запросы на каждое DNS-имя, убедитесь в том, что ответы приходят от разных приложений.

```bash
$ curl http://nginx.example.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

$ curl http://blue.nginx.example.com
<h1>I am <font color=blue>BLUE</font></h1>

$ curl http://green.nginx.example.com
<h1>I am <font color=green>GREEN</font></h1>
```

## Очистка тестового окружения

```bash
kubectl delete -f ingress-resource.yaml
kubectl delete svc nginx-deploy-main
kubectl delete svc nginx-deploy-blue
kubectl delete svc nginx-deploy-green
kubectl delete -f nginx-deploy-main.yaml -f nginx-deploy-blue.yaml -f nginx-deploy-green.yaml
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete ns ingress-nginx
```
