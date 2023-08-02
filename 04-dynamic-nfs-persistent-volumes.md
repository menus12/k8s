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

## Проверка работы динамического развертывания постоянных томов

Проверьте отсутствие каких либо pv и pvc в кластере на текущий момент, а также убедитесь в том, что директория, которая экспортируется через NFS, на текущий момент пустая.

```bash
$ kubectl get pv,pvc
No resources found
$ ls /srv/nfs/kubedata/
$ 
```

Создайте YAML-манифест ```nfs-pvc.yaml``` для PersistentVolumeClaim. Обратите внимание, что в спецификации должно быть указано ```storageClassName: managed-nfs-storage```.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  storageClassName: managed-nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
```

Примените данный манифест с помощью команды ```kubectl create -f nfs-pvc.yaml```.

Проверьте, что при создании PersistentVolumeClaim также автоматически был создан PersistenVolume. Также проверьте контент в директории NFS-сервера.

```bash
$ kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS          REASON   AGE
persistentvolume/pvc-23800bc6-dbae-422b-9ff3-e92975c76fe8   500Mi      RWX            Delete           Bound    default/pvc1   managed-nfs-storage            6s

NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
persistentvolumeclaim/pvc1   Bound    pvc-23800bc6-dbae-422b-9ff3-e92975c76fe8   500Mi      RWX            managed-nfs-storage   6s

$ ls /srv/nfs/kubedata/
default-pvc1-pvc-23800bc6-dbae-422b-9ff3-e92975c76fe8
```

Создате YAML-манифест ```busybox-pvc.yaml``` для pod'а, который будет использовать созданный pvc.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  volumes:
  - name: host-volume
    persistentVolumeClaim:
      claimName: pvc1
  containers:
  - image: busybox
    name: busybox
    command: ["/bin/sh"]
    args: ["-c", "sleep 600"]
    volumeMounts:
    - name: host-volume
      mountPath: /mydata
```

Примените данный манифест с помощью команды ```kubectl create -f busybox-pvc.yaml```.

Дождитесь пока контейнер запустится и подключитесь к нему. Создайте внутри директории /mydata файл hello с текстом world.

```bash
$ kubectl get pod busybox
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          9s

$ kubectl exec -it busybox -- /bin/sh
/ #
/ # ls /mydata
/ # echo world > /mydata/hello
/ # exit
```

Проверьте наличие данного файла в директории pv созданную автоматически на NFS-сервере.

```bash
$ cat /srv/nfs/kubedata/default-pvc1-pvc-23800bc6-dbae-422b-9ff3-e92975c76fe8/hello
world
```

Внесите изменения в YAML-манифест '''nfs-pvc.yaml''' — задайте имя pvc2 и удалите директиву storageClass. Примените созданный манифест и проверьте, что данный pvc был создан в созданном классе хранения ```managed-nfs-storage```, а также что в корневой директории NFS-сервера автоматически была создана директория для pv.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
```

```bash
$ kubectl create -f nfs-pvc.yaml
persistentvolumeclaim/pvc2 created

$ kubectl get pv,pvc
NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS          REASON   AGE
persistentvolume/pvc-23800bc6-dbae-422b-9ff3-e92975c76fe8   500Mi      RWX            Delete           Bound    default/pvc1   managed-nfs-storage            7m51s
persistentvolume/pvc-ee9a6c2a-31c2-47b5-b405-17f66ff6d244   500Mi      RWX            Delete           Bound    default/pvc2   managed-nfs-storage            18s

NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
persistentvolumeclaim/pvc1   Bound    pvc-23800bc6-dbae-422b-9ff3-e92975c76fe8   500Mi      RWX            managed-nfs-storage   7m51s
persistentvolumeclaim/pvc2   Bound    pvc-ee9a6c2a-31c2-47b5-b405-17f66ff6d244   500Mi      RWX            managed-nfs-storage   18s

$ ls /srv/nfs/kubedata/
default-pvc1-pvc-23800bc6-dbae-422b-9ff3-e92975c76fe8  default-pvc2-pvc-ee9a6c2a-31c2-47b5-b405-17f66ff6d244
```

## Очистка тестового окружения



```bash
kubectl delete -f busybox-pvc.yaml
kubectl delete pvc --all
kubectl delete -f deployment.yaml -f storageClass.yaml -f rbac.yaml
```
