## Предварительая конфигурация

* Развернут балансировщик нагрузки
* Развернут nginx ingress
* Развернут nfs client provisioner
* Установлен Helm v3

## Установка loki-stack

С помощью Helm сохраните файл со значениями для возможности его локального редактирования.

```bash
helm show values loki-stack --repo https://grafana.github.io/helm-charts > loki-stack-values.yaml
```

Отредактируйте файл loki-stack-values.yaml. 

* В директиве ```loki``` добавьте следующуюу директиву:

```yaml
  persistence:
    enabled: true
    size: 1Gi
```

* В директиве ```grafana``` измените значения ключа ```enabled``` с ```false``` на ```true```.
* В директиве ```promrtheus``` измените значения ключа ```enabled``` с ```false``` на ```true```.

Установите loki-stack с помощью Helm, используя локальный файл со значениями loki-stack-values.yaml. Все компоненты будут установлены в пространстве имен loki-stack.

```bash
helm upgrade --install loki-stack loki-stack --repo https://grafana.github.io/helm-charts --values loki-stack-values.yaml --namespace loki-stack --create-namespace
```

Проверьте, что все компоненты установлены и активны.

```bash
$ kubectl get all -n loki-stack
NAME                                                      READY   STATUS    RESTARTS   AGE
pod/loki-stack-0                                          1/1     Running   0          93s
pod/loki-stack-grafana-cc59ccc96-5qcrl                    2/2     Running   0          16s
pod/loki-stack-kube-state-metrics-86b484b9b9-q4sbh        1/1     Running   0          93s
pod/loki-stack-prometheus-alertmanager-7bc6d6b666-4s88r   2/2     Running   0          93s
pod/loki-stack-prometheus-node-exporter-nvftn             1/1     Running   0          94s
pod/loki-stack-prometheus-pushgateway-788bb6d4fd-twt9v    1/1     Running   0          93s
pod/loki-stack-prometheus-server-689b8bbc9c-vh7bn         2/2     Running   0          93s
pod/loki-stack-promtail-2sczz                             1/1     Running   0          93s
pod/loki-stack-promtail-h9szl                             1/1     Running   0          93s

NAME                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/loki-stack                            ClusterIP   10.100.57.216    <none>        3100/TCP   94s
service/loki-stack-grafana                    ClusterIP   10.103.148.190   <none>        80/TCP     16s
service/loki-stack-headless                   ClusterIP   None             <none>        3100/TCP   94s
service/loki-stack-kube-state-metrics         ClusterIP   10.105.228.86    <none>        8080/TCP   94s
service/loki-stack-memberlist                 ClusterIP   None             <none>        7946/TCP   94s
service/loki-stack-prometheus-alertmanager    ClusterIP   10.109.13.149    <none>        80/TCP     94s
service/loki-stack-prometheus-node-exporter   ClusterIP   None             <none>        9100/TCP   94s
service/loki-stack-prometheus-pushgateway     ClusterIP   10.109.9.166     <none>        9091/TCP   94s
service/loki-stack-prometheus-server          ClusterIP   10.110.170.89    <none>        80/TCP     94s

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/loki-stack-prometheus-node-exporter   1         1         1       1            1           <none>          94s
daemonset.apps/loki-stack-promtail                   2         2         2       2            2           <none>          94s

NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/loki-stack-grafana                   1/1     1            1           16s
deployment.apps/loki-stack-kube-state-metrics        1/1     1            1           94s
deployment.apps/loki-stack-prometheus-alertmanager   1/1     1            1           94s
deployment.apps/loki-stack-prometheus-pushgateway    1/1     1            1           94s
deployment.apps/loki-stack-prometheus-server         1/1     1            1           94s

NAME                                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/loki-stack-grafana-cc59ccc96                    1         1         1       16s
replicaset.apps/loki-stack-kube-state-metrics-86b484b9b9        1         1         1       93s
replicaset.apps/loki-stack-prometheus-alertmanager-7bc6d6b666   1         1         1       93s
replicaset.apps/loki-stack-prometheus-pushgateway-788bb6d4fd    1         1         1       93s
replicaset.apps/loki-stack-prometheus-server-689b8bbc9c         1         1         1       93s

NAME                          READY   AGE
statefulset.apps/loki-stack   1/1     94s
```
## Создание сервисов внешних имен и ingress resource

Так как сервис для веб-доступа к grafana расположен в отдельном пространстве имен, необходимо создать сервис внешнего имени, на который будет ссылаться ingress resource.

Создайте YAML-манифест ```extname-grafana.yaml``` для сервиса внешнего имени.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: grafana
spec:
  type: ExternalName
  externalName: loki-stack-grafana.loki-stack.svc.cluster.local
```

Примените созданный YAML-манифест и проверьте, что он появились в пространстве имен по умолчанию. Обратите внимание, что в поле EXTERNAL-IP указанно полное доменное имя сервиса.

```bash
kubectl create -f extname-grafana.yaml

$ kubectl get svc grafana
NAME      TYPE           CLUSTER-IP   EXTERNAL-IP                                       PORT(S)   AGE
grafana   ExternalName   <none>       loki-stack-grafana.loki-stack.svc.cluster.local   <none>    33s
```

Создайте YAML-манифест ```ingress-resource.yaml``` и укажите в нем созданное внешние имя для сервиса grafana. Данное правило будет использоваться для маршрутизации запросов к опубликованым сервисам по DNS-именам:

* http://grafana.example.com --> grafana:80 (--> loki-stack-grafana.loki-stack.svc.cluster.local:80)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-nginx
spec:
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 80
```

Примените созданный YAML-манифест и проверьте его параметры. Обратите внимание на ошибку в поле ```Backends``` — ```<error: endpoints not found>)``` — которая возникает из-за того, что endpoint отсутствует в данном пространстве имен.

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
  Host                 Path  Backends
  ----                 ----  --------
  grafana.example.com    /   grafana:80 (<error: endpoints "grafana" not found>)
Annotations:           <none>
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    52s (x2 over 59s)  nginx-ingress-controller  Scheduled for sync
```

## Проверка работы loki-stack

Проверьте, какой IP-адрес используется ingress-контроллером для балансировки нагрузки.

```bash
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.178.253   192.168.133.160   80:30562/TCP,443:31608/TCP   60m
ingress-nginx-controller-admission   ClusterIP      10.96.120.252    <none>            443/TCP                      60m
```

Добавьте тестовые DNS-записи в файл hosts.

```bash
192.168.133.160 grafana.example.com
```

Войдите в веб-панель Grafana по адресу ```http://grafana.example.com```, перейдите в панель Configuration -> Data Sources и обратите внимание, что источники Loki и Prometheus уже добавлены.

Для тестирования связки Prometheus и Grafana импортируйте dashboard с номером 10000 (https://grafana.com/grafana/dashboards/10000-kubernetes-cluster-monitoring-via-prometheus/).

Для тестирования связки Loki и Grafana импортируйте dashboard с номером 13639 (https://grafana.com/grafana/dashboards/13639-logs-app/).

## Очистка тестового окружения

```bash
kubectl delete -f ingress-resource.yaml -f extname-grafana.yaml
helm uninstall loki-stack --namespace loki-stack
kubectl delete ns loki-stack
```
