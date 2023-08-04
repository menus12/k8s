## Изменение параметров kube-proxy

Перед установкой metallb необходимо внести изменения в параметры ipvs конфигурации kube-proxy — значение ключа ```strictARP``` должно быть изменено с ```false``` на ```true```. Внести изменения можно вручную с помощью команды ```kubectl edit configmap -n kube-system kube-proxy``` или автоматически с помощью следующих комманд:

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | \
sed -e "s/strictARP: false/strictARP: true/" | \
kubectl apply -f - -n kube-system
```

## Установка metallb

Установка производится автоматически и использованием одного YAML-манифеста.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```

Проверьте, что создано пространство имен ```metallb-system```, а также что в нем активны все необходимые ресурсы.

```bash
$ kubectl get ns
NAME              STATUS   AGE
default           Active   5d1h
kube-flannel      Active   5d1h
kube-node-lease   Active   5d1h
kube-public       Active   5d1h
kube-system       Active   5d1h
metallb-system    Active   25s

$ kubectl get all -n metallb-system
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-595f88d88f-wvf47   1/1     Running   0          40s
pod/speaker-hggqq                 0/1     Running   0          40s
pod/speaker-n7qq4                 0/1     Running   0          40s

NAME                      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/webhook-service   ClusterIP   10.107.79.191   <none>        443/TCP   40s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/speaker   2         2         0       2            0           kubernetes.io/os=linux   40s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           41s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-595f88d88f   1         1         1       41s
```

## Конфигурация пула IP-адресов для балансировщика нагруки

Перед тем как задать параметры IP-пула для metallb, уточните в какой сети размещены активные узлы. Диапазон IP-адресов для балансировщика должен находится в этой же сети (в данном примере 192.168.133.0/24).

```bash
$ kubectl get nodes -o wide
NAME    STATUS                        ROLES           AGE    VERSION   INTERNAL-IP       EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
kub01   Ready                         control-plane   5d1h   v1.27.4   192.168.133.151   <none>        Ubuntu 23.04   6.2.0-26-generic   containerd://1.6.21
kub02   Ready                         <none>          5d1h   v1.27.4   192.168.133.152   <none>        Ubuntu 23.04   6.2.0-26-generic   containerd://1.6.21
```

Создайте YAML-манифест ```pool.yaml``` и укажите диапозон IP-адресов, которые балансировщик будет предоставлять сервисам с типом ```LoadBalancer```.

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lb-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.133.160-192.168.133.179
```

Примените данный манифест и проверьте, что ресурс создан.

```bash
kubectl create -f pool.yaml

$ kubectl describe ipaddresspools.metallb.io lb-pool -n metallb-system
Name:         lb-pool
Namespace:    metallb-system
Labels:       <none>
Annotations:  <none>
API Version:  metallb.io/v1beta1
Kind:         IPAddressPool
Metadata:
  Creation Timestamp:  2023-08-04T14:33:31Z
  Generation:          1
  Resource Version:    110477
  UID:                 76987f6b-f928-4808-af85-c41ae44d0c94
Spec:
  Addresses:
    192.168.133.160-192.168.133.179
  Auto Assign:       true
  Avoid Buggy I Ps:  false
Events:              <none>
```

Создайте YAML-манифест ```L2Advertisement.yaml``` для сетевых оповещений на канальном уровне и в спецификации укажите созданный IP-пул.

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - lb-pool
```

Примените созданный манифест и проверьте, что ресурс создан.

```bash
kubectl create -f L2Advertisement.yaml
  
$ kubectl describe l2advertisements.metallb.io example -n metallb-system
Name:         example
Namespace:    metallb-system
Labels:       <none>
Annotations:  <none>
API Version:  metallb.io/v1beta1
Kind:         L2Advertisement
Metadata:
  Creation Timestamp:  2023-08-04T14:36:01Z
  Generation:          1
  Resource Version:    110761
  UID:                 12f77425-98d9-4d68-ae7c-38ff74c0bb11
Spec:
  Ip Address Pools:
    lb-pool
Events:  <none>
```

## Проверка работы балансировщика нагрузки

Создайте deployment с контейнером nginx по умолчанию и опубликуйте сервис с типом ```LoadBalancer```.

```bash
kubectl create deploy nginx --image nginx
kubectl expose deploy nginx --port 80 --type LoadBalancer
```

Проверьте, что контейнер запущен и сервис опубликован. Обратите внимание, что сервису назначен ```EXTERNAL-IP``` из IP-пула ```lb-pool```.

```bash
$ kubectl get all
NAME                                          READY   STATUS    RESTARTS      AGE
pod/nginx-77b4fdf86c-wwd29                    1/1     Running   0             109s

NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP       PORT(S)                        AGE
service/kubelet      ClusterIP      None          <none>            10250/TCP,10255/TCP,4194/TCP   25h
service/kubernetes   ClusterIP      10.96.0.1     <none>            443/TCP                        5d2h
service/nginx        LoadBalancer   10.102.39.6   192.168.133.160   80:30695/TCP                   7s

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx                    1/1     1            1           109s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-77b4fdf86c                    1         1         1       109s
```

Протестируйте доступность сервиса через внешний IP-адрес балансировщика нагрузки.

```bash
$ curl 192.168.133.160
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
kubectl delete svc nginx
kubectl delete deploy nginx
kubectl delete -f L2Advertisement.yaml
kubectl delete -f pool.yaml
kubectl delete -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml
```
