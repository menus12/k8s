## Установка и запуск NFS-сервера

На узле, который выполняет роль NFS-сервера (в данном примере kub01) выполните команды для установки пакетов и запуска сервиса.

```bash
sudo apt install -y nfs-kernel-server
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
```

Создайте директорию, в которой будут размещаться тома для монтрования.

```bash
sudo mkdir -p /srv/nsf/kubedata
sudo chmod -R 777 /srv/nsf/kubedata
```

Задайте параметры экспорта данной директории через NFS-сервер и примените изменения.

```bash
sudo echo "/srv/nfs/kubedata *(rw,sync,no_subtree_check,insecure)" >> /etc/exports
sudo exportfs -rav
```

Проверьте экспорт директории на сервере.

```bash
$ sudo exportfs -v
/srv/nfs/kubedata
                <world>(sync,wdelay,hide,no_subtree_check,sec=sys,rw,insecure,root_squash,no_all_squash)

$ showmount -e
Export list for kub01:
/srv/nfs/kubedata *
```

На узлах, на которых будут запускаться Pod'ы, установите необходимые компоненты для работы с NFS и проверьте возмжность монтирования директории через NFS.

```bash
sudo apt install -y nfs-common
sudo mount -t nfs kub01:/srv/nfs/kubedata /mnt
```

На NFS-сервере создайте файл и прочитайте его на узле, на котором примонтрована NFS-директория, затем удалите его и размонтируйте NFS-директорию

```bash
kub01:~$ echo hello > /srv/nfs/kubedata/hello
kub02:~$ cat /mnt/hello
hello
kub02:~$ rm /mnt/hello
kub02:~$ sudo umount /mnt
```

## Создание YAML манифестов

Создайте YAML-манифест ```rbac.yaml```. В данном манифесте будут описаны следующие сущности:

* Роль — с правилами "get", "list", "watch", "create", "update", "patch" для всех endpoints
* Сервисный аккаунт
* Привязка роли к сервисному аккаунту
* Роль кластера с правилами
  * "get", "list", "watch", "create", "delete" для persistentvolumes
  * "get", "list", "watch", "update" для persistentvolumeclaims
  * "get", "list", "watch" для storageclasses
  * "create", "update", "patch" для events
* Привязка роли кластера к сервисному аккаунту

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-client-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

Создайте YAML-манифеси ```storageClass.yaml```. Данный класс хранения будет использоваться по умолчанию.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: example.com/nfs
parameters:
  archiveOnDelete: "false"
```

Создайте YAML-манифест ```deployment.yaml```. Данный манифест развернет на стороне кластера pod, который будет отвечать за автоматическое монтирование томов для оатсльных pod'ов через класс хранения по умолчанию. В данном примере в качестве NFS-сервера используется ```kub01```.

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: example.com/nfs
            - name: NFS_SERVER
              value: kub01
            - name: NFS_PATH
              value: /srv/nfs/kubedata
      volumes:
        - name: nfs-client-root
          nfs:
            server: kub01
            path: /srv/nfs/kubedata
```

Примените созданные YAML-манифесты. 

```bash
kubectl create -f rbac.yaml
kubectl create -f storageClass.yaml
kubectl create -f deployment.yaml
```

## Установка Helm

Для автоматической установки Helm версии 3 выполните следующую команду.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Проверьте, что установка прошла успешно.

```bash
$ helm version --short
v3.12.2+g1e210a2
```

## Установка Prometheus

Добавьте репозиторий Prometheus Community в Helm.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

$ helm repo list
NAME                    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
```

Скачайте файл values со значениями Prometheus для возможности локального редактирования.

```bash
helm show values prometheus-community/prometheus > prometheus-values.yaml
```

Отредактируйте локальный файл ```prometheus-values.yaml```. 

* В директиве ```server.service``` измените значение ключа ```type``` с ```ClusterIP``` на ```NodePort```
* Добавьте ключ ```nodePort``` со значением ```30080```.

В итоге, директива ```server.service``` должна выглядеть так:

```yaml
  service:
    ## If false, no Service will be created for the Prometheus server
    ##
    enabled: true

    annotations: {}
    labels: {}
    clusterIP: ""

    ## List of IP addresses at which the Prometheus server service is available
    ## Ref: https://kubernetes.io/docs/concepts/services-networking/service/#external-ips
    ##
    externalIPs: []

    loadBalancerIP: ""
    loadBalancerSourceRanges: []
    servicePort: 80
    sessionAffinity: None
    nodePort: 30080
    type: NodePort
```

Установите Prometheus через Helm в отдельном пространстве имен ```prometheus```.

```bash
helm install prometheus prometheus-community/prometheus --values prometheus-values.yaml --namespace prometheus --create-namespace
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
pod/prometheus-alertmanager-0                            1/1     Running   0          2m12s
pod/prometheus-kube-state-metrics-69c8887cfd-g92qm       1/1     Running   0          2m12s
pod/prometheus-prometheus-node-exporter-hk4hn            0/1     Pending   0          2m12s
pod/prometheus-prometheus-node-exporter-jzh5x            1/1     Running   0          2m12s
pod/prometheus-prometheus-node-exporter-r4rh5            1/1     Running   0          2m12s
pod/prometheus-prometheus-pushgateway-79ff799669-krz8k   1/1     Running   0          2m12s
pod/prometheus-server-b978f5586-m9s6v                    2/2     Running   0          2m12s

NAME                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/prometheus-alertmanager               ClusterIP   10.107.230.226   <none>        9093/TCP       2m12s
service/prometheus-alertmanager-headless      ClusterIP   None             <none>        9093/TCP       2m12s
service/prometheus-kube-state-metrics         ClusterIP   10.102.98.120    <none>        8080/TCP       2m12s
service/prometheus-prometheus-node-exporter   ClusterIP   10.103.230.180   <none>        9100/TCP       2m12s
service/prometheus-prometheus-pushgateway     ClusterIP   10.105.168.242   <none>        9091/TCP       2m12s
service/prometheus-server                     NodePort    10.98.85.233     <none>        80:30080/TCP   2m12s

NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/prometheus-prometheus-node-exporter   3         3         2       3            2           kubernetes.io/os=linux   2m12s

NAME                                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-kube-state-metrics       1/1     1            1           2m12s
deployment.apps/prometheus-prometheus-pushgateway   1/1     1            1           2m12s
deployment.apps/prometheus-server                   1/1     1            1           2m12s

NAME                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-kube-state-metrics-69c8887cfd       1         1         1       2m12s
replicaset.apps/prometheus-prometheus-pushgateway-79ff799669   1         1         1       2m12s
replicaset.apps/prometheus-server-b978f5586                    1         1         1       2m12s

NAME                                       READY   AGE
statefulset.apps/prometheus-alertmanager   1/1     2m12s
```

Проверьте, что pv были автоматически созданы в нужном классе хранения.

```bash
$ kubectl get pvc -n prometheus
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
prometheus-server                   Bound    pvc-150357bb-0f8d-4053-a8c6-ebb68defd313   8Gi        RWO
managed-nfs-storage   3m41s
storage-prometheus-alertmanager-0   Bound    pvc-05c1b98d-d6ad-4c43-8b25-142bad80d65e   2Gi        RWO
managed-nfs-storage   3m41s

$ ls /srv/nfs/kubedata/
prometheus-prometheus-server-pvc-150357bb-0f8d-4053-a8c6-ebb68defd313
prometheus-storage-prometheus-alertmanager-0-pvc-05c1b98d-d6ad-4c43-8b25-142bad80d65e
```

Проверьте через веб-браузер, что веб-сервис Prometheus доступен из внешней сети по HTTP на порту 30080.

```bash
$ curl http://kub02:30080/
<a href="/graph">Found</a>.
```

## Установка Grafana

Добавьте репозиторий Grafana в Helm.

```bash
helm repo add grafana https://grafana.github.io/helm-charts

$ helm repo list
NAME                    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
grafana                 https://grafana.github.io/helm-charts
```

Скачайте файл values со значениями Grafana для возможности локального редактирования.

```bash
helm show values grafana/grafana > grafana-values.yaml
```

Отредактируйте локальный файл ```grafana-values.yaml```.

* В директиве ```service``` измените значение ключа ```type``` с ```ClusterIP``` на ```NodePort```
* Добавьте ключ ```nodePort``` со значением ```30180```
* Раскоментируйте ключ ```adminPassword```, оставьте пароль по умолчанию
* В директиве ```persistence``` измените значение ключа ```enabled``` с ```false``` на ```true```
* Значение кюча ```size``` измените на ```2Gi```
* В директиве ```initChownData``` измените значение ключа ```enabled``` с ```true``` на ```false```

В итоге, измененные директивы должны выглядеть так:

```yaml
service:
  enabled: true
  type: NodePort
  port: 80
  targetPort: 3000
  nodePort: 30180

~~~~~

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
helm install grafana grafana/grafana --values grafana-values.yaml --namespace
grafana --create-namespace
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
pod/grafana-567c957b5-wzhls   1/1     Running   0          53m

NAME              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/grafana   NodePort   10.107.24.104   <none>        80:30180/TCP   53m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   1/1     1            1           53m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-567c957b5   1         1         1       53m
```

Проверьте с помощью веб-браузера, что веб-сервис Grafana доступен из внешних сетей на порту 30180.

```bash
$ curl http://kub02:30180
<a href="/login">Found</a>.
```

Войдите в веб-панель Grafana, добавьте Data Source, выберите Prometheus. В качестве ```Prometheus server URL``` укажите адрес и порт сервиса ```prometheus-server```, который доступен изнутри кластера (в данном примере http://10.98.85.233:80).

```bash
$ kubectl get svc -n prometheus
NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
prometheus-alertmanager               ClusterIP   10.107.230.226   <none>        9093/TCP       50m
prometheus-alertmanager-headless      ClusterIP   None             <none>        9093/TCP       50m
prometheus-kube-state-metrics         ClusterIP   10.102.98.120    <none>        8080/TCP       50m
prometheus-prometheus-node-exporter   ClusterIP   10.103.230.180   <none>        9100/TCP       50m
prometheus-prometheus-pushgateway     ClusterIP   10.105.168.242   <none>        9091/TCP       50m
prometheus-server                     NodePort    10.98.85.233     <none>        80:30080/TCP   50m
```

Для тестирования связки Prometheus и Grafana импортируйте dashboard с номером 10000 (https://grafana.com/grafana/dashboards/10000-kubernetes-cluster-monitoring-via-prometheus/).

## Очистка тестового окружения

```bash
helm uninstall grafana  --namespace grafana
helm uninstall prometheus  --namespace prometheus
kubectl delete pvc --all
kubectl delete -f deployment.yaml -f storageClass.yaml -f rbac.yaml
```
