## Предварительая конфигурация

* Развернут балансировщик нагрузки
* Развернут nginx ingress
* Развернут nfs client provisioner
* Установлен Helm v3

## Установка Prometheus

Установите Prometheus через Helm в отдельном пространстве имен ```prometheus```.

```bash
helm upgrade --install prometheus prometheus --repo https://prometheus-community.github.io/helm-charts --namespace prometheus --create-namespace
```

Проверьте, что helm-chart в статусе ```deployed```.

```bash
$ helm list -n prometheus
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART             APP VERSION
prometheus      prometheus      1               2023-08-03 10:42:21.992172165 +0000 UTC deployed        prometheus-23.2.0 v2.46.0
```

Проверьте, что все ресурсы созданы и активны.

```bash
$ kubectl get all -n prometheus
NAME                                                     READY   STATUS    RESTARTS   AGE
pod/prometheus-alertmanager-0                            1/1     Running   0          40m
pod/prometheus-kube-state-metrics-69c8887cfd-sxvtv       1/1     Running   0          40m
pod/prometheus-prometheus-node-exporter-kf8t7            1/1     Running   0          40m
pod/prometheus-prometheus-node-exporter-lx5p4            1/1     Running   0          40m
pod/prometheus-prometheus-pushgateway-79ff799669-pmml2   1/1     Running   0          40m
pod/prometheus-server-b978f5586-vqskt                    2/2     Running   0          40m

NAME                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/prometheus-alertmanager               ClusterIP   10.96.34.0       <none>        9093/TCP   40m
service/prometheus-alertmanager-headless      ClusterIP   None             <none>        9093/TCP   40m
service/prometheus-kube-state-metrics         ClusterIP   10.96.133.109    <none>        8080/TCP   40m
service/prometheus-prometheus-node-exporter   ClusterIP   10.107.242.229   <none>        9100/TCP   40m
service/prometheus-prometheus-pushgateway     ClusterIP   10.106.126.121   <none>        9091/TCP   40m
service/prometheus-server                     ClusterIP   10.101.220.21    <none>        80/TCP     40m

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/prometheus-prometheus-node-exporter   2         2         2       2            2           kubernetes.io/os=linux   40m

NAME                                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-kube-state-metrics       1/1     1            1           40m
deployment.apps/prometheus-prometheus-pushgateway   1/1     1            1           40m
deployment.apps/prometheus-server                   1/1     1            1           40m

NAME                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-kube-state-metrics-69c8887cfd       1         1         1       40m
replicaset.apps/prometheus-prometheus-pushgateway-79ff799669   1         1         1       40m
replicaset.apps/prometheus-server-b978f5586                    1         1         1       40m

NAME                                       READY   AGE
statefulset.apps/prometheus-alertmanager   1/1     40m
```

Проверьте, что pv были автоматически созданы в нужном классе хранения.

```bash
$ kubectl get pvc -n prometheus
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
prometheus-server                   Bound    pvc-6d6c640b-9e4a-4105-8706-30fd2ab90b39   8Gi        RWO            managed-nfs-storage   41m
storage-prometheus-alertmanager-0   Bound    pvc-e071a030-1043-49fe-b3d4-270a02b4f4c8   2Gi        RWO            managed-nfs-storage   41m

$ ls /srv/nfs/kubedata/
prometheus-prometheus-server-pvc-150357bb-0f8d-4053-a8c6-ebb68defd313
prometheus-storage-prometheus-alertmanager-0-pvc-05c1b98d-d6ad-4c43-8b25-142bad80d65e
```

## Установка Grafana

Скачайте файл values со значениями Grafana для возможности локального редактирования.

```bash
helm show values grafana --repo https://grafana.github.io/helm-charts > grafana-values.yaml
```

Отредактируйте локальный файл ```grafana-values.yaml```.

* Раскоментируйте ключ ```adminPassword```, оставьте пароль по умолчанию
* В директиве ```persistence``` измените значение ключа ```enabled``` с ```false``` на ```true```
* Значение кюча ```size``` измените на ```2Gi```
* В директиве ```initChownData``` измените значение ключа ```enabled``` с ```true``` на ```false```

В итоге, измененные директивы должны выглядеть так:

```yaml
persistence:
  type: pvc
  enabled: true
  #storageClassName: default
  accessModes:
    - ReadWriteOnce
  size: 2Gi
  # annotations: {}
  finalizers:
    - kubernetes.io/pvc-protection

~~~~~

initChownData:
  ## If false, data ownership will not be reset at startup
  ## This allows the grafana-server to be run with an arbitrary user
  ##
  enabled: false
```

Установите Grafana с помощью Helm в отдельном пространстве имен ```grafana```.

```bash
helm upgrade --install grafana grafana --repo https://grafana.github.io/helm-charts --values grafana-values.yaml --namespace grafana --create-namespace
```

Проверьте, что helm-chart в статусе ```deployed```.

```bash
$ helm list -n grafana
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
grafana grafana         1               2023-08-03 11:30:03.187605954 +0000 UTC deployed        grafana-6.58.7  10.0.3
```

Проверьте, что все ресурсы созданы и активны.

```bash
$ kubectl get all -n grafana
NAME                          READY   STATUS    RESTARTS   AGE
pod/grafana-567c957b5-rm4xz   1/1     Running   0          41m

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/grafana   ClusterIP   10.111.35.166   <none>        80/TCP    41m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   1/1     1            1           41m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-567c957b5   1         1         1       41m
```

## Создание сервисов внешних имен и ingress resource

Так как сервисы для веб-доступа к prometheus и grafana расположены в отдельных пространствах имен, необходимо создать сервисы внешних имен, на которые будет ссылаться ingress resource.

Создайте YAML-манифесты для сервисов внешних имен.

* extname-prometheus.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
  name: prometheus
spec:
  type: ExternalName
  externalName: prometheus-server.prometheus.svc.cluster.local
```

* extname-grafana.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
  name: grafana
spec:
  type: ExternalName
  externalName: grafana.grafana.svc.cluster.local
```

Примените созданные YAML-манифесты и проверьте, что они появились в пространстве имен по умолчанию. Обратите внимание, что в поле EXTERNAL-IP указанны полные доменные имена соответствующих сервисов.

```bash
kubectl create -f extname-prometheus.yaml -f extname-grafana.yaml

$ kubectl get svc
NAME         TYPE           CLUSTER-IP   EXTERNAL-IP                                      PORT(S)                        AGE
grafana      ExternalName   <none>       grafana.grafana.svc.cluster.local                <none>                         34m
kubelet      ClusterIP      None         <none>                                           10250/TCP,10255/TCP,4194/TCP   2d
kubernetes   ClusterIP      10.96.0.1    <none>                                           443/TCP                        6d
prometheus   ExternalName   <none>       prometheus-server.prometheus.svc.cluster.local   <none>                         31m
```

Создайте YAML-манифест ```ingress-resource.yaml``` и укажите в нем созданные внешние имена для сервисов prometheus и grafana. Данные правида будут использоваться для маршрутизации запросов к опубликованым сервисам по DNS-именам:

* http://prometheus.example.com --> prometheus:80 (--> prometheus-server.prometheus.svc.cluster.local:80)
* http://grafana.example.com --> grafana:80 (--> grafana.grafana.svc.cluster.local:80)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-nginx
spec:
  rules:
  - host: prometheus.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus
            port:
              number: 80
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

Примените созданный YAML-манифест и проверьте его параметры. Обратите внимание на ошибки в поле ```Backends``` — ```<error: endpoints not found>)``` — которые возникают из-за того, что endpoint'ы отсутствуют в данном пространстве имен.

```bash
kubectl create -f ingress-resource.yaml

$ kubectl describe ing ingress-resource-nginx
Name:             ingress-resource-nginx
Labels:           <none>
Namespace:        default
Address:
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host                    Path  Backends
  ----                    ----  --------
  prometheus.example.com    /   prometheus:80 (<error: endpoints "prometheus" not found>)
  grafana.example.com       /   grafana:80 (<error: endpoints "grafana" not found>)
Annotations:              <none>
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  Sync    8s    nginx-ingress-controller  Scheduled for sync
```

## Проверка работы Prometheus и Grafana

Проверьте, какой IP-адрес используется ingress-контроллером для балансировки нагрузки.

```bash
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP       PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.178.253   192.168.133.160   80:30562/TCP,443:31608/TCP   60m
ingress-nginx-controller-admission   ClusterIP      10.96.120.252    <none>            443/TCP                      60m
```

Добавьте тестовые DNS-записи в файл hosts.

```bash
192.168.133.160 prometheus.example.com
192.168.133.160 grafana.example.com
```

Войдите в веб-панель Grafana по адресу ```http://grafana.example.com```, добавьте Data Source, выберите Prometheus. В качестве ```Prometheus server URL``` укажите FQDN сервиса ```prometheus-server```, который доступен изнутри кластера — ```prometheus-server.prometheus.svc.cluster.local```.

Для тестирования связки Prometheus и Grafana импортируйте dashboard с номером 10000 (https://grafana.com/grafana/dashboards/10000-kubernetes-cluster-monitoring-via-prometheus/).

Также проверьте, что веб-панель Prometheus доступна через веб-интерфейс по адресу ```http://prometheus.example.com```.

## Очистка тестового окружения

```bash
kubectl delete -f ingress-resource.yaml -f extname-prometheus.yaml -f extname-grafana.yaml
helm uninstall grafana --namespace grafana
helm uninstall prometheus --namespace prometheus
kubectl delete ns prometheus
kubectl delete ns grafana
```
