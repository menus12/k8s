## Предварительая конфигурация

* Развернут NFS-сервер

## Создание Persistemt Volume и Persistent Volume Claim

Создайте YAML манифест для Persistent Volume ```nfs-pv.yaml```.

```bash
cat <<EOF > nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-pv1
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: kub01
    path: "/srv/nfs/kubedata"
EOF
```

Примените данный манифест с помощью ```kubectl```.

```bash
kubectl create -f nfs-pv.yaml
```

Проверьте, что PV создан.

```bash
$ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-nfs-pv1   1Gi        RWX            Retain           Available           manual                  24s
```

Создайте YAML манифест для Persistent Volume Claim ```nfs-pvc.yaml```.

```bash
cat <<EOF > nfs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-pv1
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Mi
EOF
```

Примените данный манифест с помощью ```kubectl```.

```bash
kubectl create -f nfs-pvc.yaml
```

Проверьте, что PVC создан.

```bash
$ kubectl get pv,pvc
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
persistentvolume/pv-nfs-pv1   1Gi        RWX            Retain           Bound    default/pvc-nfs-pv1   manual                  14m

NAME                                STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/pvc-nfs-pv1   Bound    pv-nfs-pv1   1Gi        RWX            manual         7s
```

## Развертывание приложения с использованием созданных PV и PVC

На NFS-сервере создайте файл index.html, который будет использоваться как стартовая страница для Nginx.

```bash
echo "<h1>Hello World</h1>" > /srv/nfs/kubedata/index.html
```

Создайте deployment-манифест для Nginx с указанием созданного PVC для монтирования тома. Директория ```/srv/nfs/kubedata``` NFS-сервера будет смонтирована в  ```/usr/share/nginx/html``` внутри контейнера.

```bash
cat <<EOF > nfs-ngnix-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: nginx
  name: nginx-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      volumes:
      - name: www
        persistentVolumeClaim:
          claimName: pvc-nfs-pv1
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
EOF
```

Примените данный манифест с помощью ```kubectl```.

```bash
kubectl create -f nfs-ngnix-demo.yaml
```

Дождитесь, пока контейнер запустится.

```bash
$ kubectl get pod
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-559c7cff6c-nsmp6   1/1     Running   0          39s
```

## Проверка работы PV и PVC

Подключитесь к интерпритатору контейнера и проверьте наличие файла ```index.html``` в директории ```/usr/share/nginx/html```.

```bash
$ kubectl exec -it nginx-deploy-559c7cff6c-nsmp6 -- /bin/sh
# ls /usr/share/nginx/html
index.html
# cat /usr/share/nginx/html/index.html
<h1>Hello World</h1>
# exit
```

Опубликуйте сервис для nginx-deploy, чтобы иметь возможность получить доступ к веб-серверу из вне.

```bash
kubectl expose deploy nginx-deploy --port 80 --type NodePort
```

Проверьте, на каком порту работает сервис и на каком узле запущен контейнер с Nginx, сделайте http-запрос с помощью ```curl```.

```bash
$ kubectl get service
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP        2d1h
nginx-deploy   NodePort    10.111.80.219   <none>        80:30477/TCP   11s

$ kubectl describe pod nginx-deploy | grep Node:
Node:             kub02/192.168.133.152

$ curl http://kub02:30477
<h1>Hello World</h1>
```

На NFS-сервере измените содержание файла index.html и снова сделайте проверочный http-запрос.

```bash
$ echo "<h1>Another Heading</h1>" > /srv/nfs/kubedata/index.html
$ curl http://kub02:30477
<h1>Another Heading</h1>
```

## Очистка окружения

Выполните следующие команды для очистки тестового окружения.

```bash
kubectl delete svc nginx-deploy
kubectl delete deploy nginx-deploy
kubectl delete pvc pvc-nfs-pv1
kubectl delete pv pv-nfs-pv1
```
