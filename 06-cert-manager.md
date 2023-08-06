## Предварительая конфигурация

* Развернут балансировщик нагрузки
* Развернут nginx ingress

## Установка cert manager

Установите cert manager из YAML-манифеста в официальном репозитории. По умолчанию все компоненты будут созданы в пространстве имен ```cert-manager```. Проверьте что все компоненты установлены и активны.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml

$ kubectl get all -n cert-manager
NAME                                          READY   STATUS    RESTARTS   AGE
pod/cert-manager-7476c8fcf4-sr52q             1/1     Running   0          40s
pod/cert-manager-cainjector-bdd866bd4-jl92v   1/1     Running   0          40s
pod/cert-manager-webhook-5655dcfb4b-dtxrd     1/1     Running   0          40s

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.109.124.49    <none>        9402/TCP   40s
service/cert-manager-webhook   ClusterIP   10.107.156.196   <none>        443/TCP    40s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           40s
deployment.apps/cert-manager-cainjector   1/1     1            1           40s
deployment.apps/cert-manager-webhook      1/1     1            1           40s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-7476c8fcf4             1         1         1       40s
replicaset.apps/cert-manager-cainjector-bdd866bd4   1         1         1       40s
replicaset.apps/cert-manager-webhook-5655dcfb4b     1         1         1       40s
```

## Развертывание тестового приложения

Разверните веб-сервер nginx и опубликуйте веб-сервис.

```bash
kubectl create deploy nginx --image nginx
kubectl expose deploy nginx --port 80 

$ kubectl get all
NAME                                          READY   STATUS    RESTARTS       AGE
pod/nginx-77b4fdf86c-x2vrt                    1/1     Running   0              2m7s

NAME                 TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                        AGE
service/kubelet      ClusterIP   None          <none>        10250/TCP,10255/TCP,4194/TCP   2d23h
service/kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP                        6d23h
service/nginx        ClusterIP   10.106.9.80   <none>        80/TCP                         115s

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx                    1/1     1            1           2m7s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-77b4fdf86c                    1         1         1       2m7s
```

## Создание эмитента сертификатов

Создайте YAML-манифест ```ClusterIssuer.yaml```, в котором будет определен эмитент сертификатов для кластера. Обратите внимание, что сервер ```https://acme-staging-v02.api.letsencrypt.org/directory``` используется только для выдачи самоподписанных сертификатов в тестовых целях.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: hello@hello.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
```

Примените созданный YAML-манифест и проверьте параметры созданного эмитента.

```bash
kubectl create -f ClusterIssuer.yaml

$ kubectl describe clusterissuers.cert-manager.io
Name:         letsencrypt-staging
Namespace:
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         ClusterIssuer
Metadata:
  Creation Timestamp:  2023-08-06T12:03:32Z
  Generation:          1
  Resource Version:    160876
  UID:                 32516067-e031-47d9-8b25-fec164509004
Spec:
  Acme:
    Email:            hello@hello.com
    Preferred Chain:
    Private Key Secret Ref:
      Name:  letsencrypt-staging
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Class:  nginx
Status:
  Acme:
    Last Private Key Hash:  LL0N8rA0OJB0I65LNcciG8PdqMzr5HpBUv2U8oQ/dX0=
    Last Registered Email:  hello@hello.com
    Uri:                    https://acme-staging-v02.api.letsencrypt.org/acme/acct/113734294
  Conditions:
    Last Transition Time:  2023-08-06T12:03:33Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

## Создание Ingress Resource

Создайте YAML-манифест ```ingress-resource.yaml``` в котором будут определены правила маршрутизации запросов по DNS-имени ```nginx.example.com``` на опубликованный сервис ```nginx```, а также указан эмитент сертификатов.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  tls:
  - hosts:
    - nginx.example.com
    secretName: letsencrypt-staging
  rules:
  - host: nginx.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

Примените созданный YAML-манифест и проверьте параметры созданных правил.

```bash
kubectl create -f ingress-resource.yaml

$ kubectl get ing
NAME               CLASS   HOSTS               ADDRESS   PORTS     AGE
ingress-resource   nginx   nginx.example.com             80, 443   28s

$ kubectl describe ing ingress-resource
Name:             ingress-resource
Labels:           <none>
Namespace:        default
Address:          192.168.133.160
Ingress Class:    nginx
Default backend:  <default>
TLS:
  letsencrypt-staging terminates nginx.example.com
Rules:
  Host               Path  Backends
  ----               ----  --------
  nginx.example.com
                     /   nginx:80 (10.244.1.88:80)
Annotations:         cert-manager.io/cluster-issuer: letsencrypt-staging
Events:
  Type    Reason             Age                From                       Message
  ----    ------             ----               ----                       -------
  Normal  CreateCertificate  52s                cert-manager-ingress-shim  Successfully created Certificate "letsencrypt-staging"
  Normal  Sync               19s (x2 over 52s)  nginx-ingress-controller   Scheduled for sync
```

## Проверка работы приложения с выданным сертификатом

Проверьте, какой IP-адрес используется ingress-контроллером для балансировки нагрузки.

```bash
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.178.253   192.168.133.160   80:30562/TCP,443:31608/TCP   24h
ingress-nginx-controller-admission   ClusterIP      10.96.120.252    <none>            443/TCP                      24h
```

Добавьте тестовые DNS-записи в файл hosts.

```bash
192.168.133.160 nginx.example.com
```

В веб-браузере перейдите по адресу ```http://nginx.example.com```. Обратите внимание, что происходит автоматический редирект на ```https://nginx.example.com``` и появляется сообщение о недоверенном сертификате (т.к. он является самоподписанным). При игнорировании недеверия к сертификату, откроется стартовая страница nginx.

```bash
$ curl http://nginx.example.com
<html>
<head><title>308 Permanent Redirect</title></head>
<body>
<center><h1>308 Permanent Redirect</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

```bash
$ curl -L http://nginx.example.com
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html
```

```bash
$ curl -Lk http://nginx.example.com
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
```

## Очистка тестового окружения

```bash
kubectl delete -f ingress-resource.yaml -f ClusterIssuer.yaml
kubectl delete svc nginx
kubectl delete deploy nginx
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```
