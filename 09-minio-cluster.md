## Предварительая конфигурация

* Развернут балансировщик нагрузки
* Развернут nginx ingress
* Развернут любой storage class

## Установка minio

Создайте YAML-манифест ```secret.yaml``` где будут указаны учетные данные от кластера minio.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio
  labels:
    app.kubernetes.io/name: minio
    app.kubernetes.io/instance: minio
type: Opaque
stringData:
  user: "admin"
  password: "password"
```

Создайте YAML-манифест ```sts.yaml``` где будут указаны сервисы и stateful set.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: minio-headless
  labels:
    app.kubernetes.io/name: minio
    app.kubernetes.io/instance: minio
spec:
  type: ClusterIP
  clusterIP: None
  ports:
    - name: minio
      port: 9000
      targetPort: 9000
    - name: minio-web
      port: 9001
      targetPort: 9001
  publishNotReadyAddresses: true
  selector:
    app.kubernetes.io/name: minio
    app.kubernetes.io/instance: minio
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  labels:
    app.kubernetes.io/name: minio
    app.kubernetes.io/instance: minio
spec:
  type: ClusterIP
  ports:
    - name: minio
      port: 9000
      targetPort: 9000
  selector:
    app.kubernetes.io/name: minio
    app.kubernetes.io/instance: minio
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: minio
  labels:
    app.kubernetes.io/name: minio
    app.kubernetes.io/instance: minio
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: minio
      app.kubernetes.io/instance: minio
  serviceName: minio-headless
  replicas: 2
  podManagementPolicy: Parallel
  template:
    metadata:
      labels:
        app.kubernetes.io/name: minio
        app.kubernetes.io/instance: minio
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/minio/v2/metrics/cluster"
        prometheus.io/port: "9000"
    spec:
      securityContext:
        fsGroup: 1001
      initContainers:
        - name: init-configs
          image: busybox:1.33.0
          imagePullPolicy: "IfNotPresent"
          command:
            - sh
            - -c
            - |
              if [ ! -d /data-0/data ]; then
                mkdir /data-0/data
                chown 1001:1001 /data-0/data
              fi
              if [ ! -d /data-1/data ]; then
                mkdir /data-1/data
                chown 1001:1001 /data-1/data
              fi
          volumeMounts:
            - name: data-0
              mountPath: /data-0
            - name: data-1
              mountPath: /data-1
      containers:
        - name: minio
          # image: quay.io/bitnami/minio:2021.6.17-debian-10-r14
          ## Fix to new container version
          image: bitnami/minio:2023.5.4-debian-11-r1
          imagePullPolicy: "IfNotPresent"
          securityContext:
            runAsNonRoot: true
            runAsUser: 1001
          env:
            - name: BITNAMI_DEBUG
              value: "false"
            - name: MINIO_DISTRIBUTED_MODE_ENABLED
              value: "yes"
            - name: MINIO_DISTRIBUTED_NODES
              value: "minio-{0...1}.minio-headless.default.svc.cluster.local/data-{0...1}/data"
            - name: MINIO_SCHEME
              value: "http"
            - name: MINIO_FORCE_NEW_KEYS
              value: "no"
            - name: MINIO_ROOT_USER # MINIO_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: minio
                  key: user
            - name: MINIO_ROOT_PASSWORD # MINIO_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: minio
                  key: password
            - name: MINIO_SKIP_CLIENT
              value: "yes"
            - name: MINIO_BROWSER
              value: "on"
            - name: MINIO_PROMETHEUS_AUTH_TYPE
              value: "public"
          ports:
            - name: minio
              containerPort: 9000
              protocol: TCP
            - name: minio-web
              containerPort: 9001
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /minio/health/live
              port: minio
              scheme: "HTTP"
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 5
          readinessProbe:
            tcpSocket:
              port: minio
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 1
            successThreshold: 1
            failureThreshold: 5
          resources:
            limits: {}
            requests: {}
          volumeMounts:
            - mountPath: /data-0
              name: data-0
            - mountPath: /data-1
              name: data-1
      volumes:
        - name: data-0
          emptyDir: {}
        - name: data-1
          emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "local-path"
      resources:
        requests:
          storage: 1Gi
```

Создайте YAML-манифест ```ingress-resource.yaml``` для публикации веб-интерфейса minio.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-nginx
spec:
  ingressClassName: nginx
  rules:
  - host: minio.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: minio-headless
            port:
              number: 9001
```

Примените созданные манифесты и проверьте, что все ресурсы активны.

```bash
kubectl create -f secret.yaml -f sts.yaml -f ingress-resource.yaml

$ kubectl get all
NAME                                          READY   STATUS      RESTARTS       AGE
pod/minio-0                                   1/1     Running     0              34m
pod/minio-1                                   1/1     Running     0              34m

NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                        AGE
service/kubelet          ClusterIP   None            <none>        10250/TCP,10255/TCP,4194/TCP   4d17h
service/kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP                        8d
service/minio            ClusterIP   10.105.33.194   <none>        9000/TCP                       34m
service/minio-headless   ClusterIP   None            <none>        9000/TCP,9001/TCP              34m

NAME                     READY   AGE
statefulset.apps/minio   2/2     34m

$ kubectl describe ing
Name:             ingress-resource-nginx
Labels:           <none>
Namespace:        default
Address:          192.168.133.160
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host               Path  Backends
  ----               ----  --------
  minio.example.com
                     /   minio-headless:9001 (10.244.1.190:9001,10.244.1.191:9001)
Annotations:         <none>
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    34m (x2 over 35m)  nginx-ingress-controller  Scheduled for sync
```

## Проверка работы кластера minio

Создайте YAML-манифест ```minio-init.yaml``` для контейнера, который подключится к класеру и создаст bucket, после чего завершит свою работу.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: minio-mc
spec:
  containers:
  - name: minio-mc
    image: minio/mc
    command: ['sh', '-c', 'mc alias set minio http://minio-headless:9000 admin password && mc mb minio/test-bucket']
  restartPolicy: Never
```

Примените созданный манифест и проверьте логи его выполнения.

```bash
kubectl create -f minio-init.yaml

$ kubectl logs minio-mc
Added `minio` successfully.
Bucket created successfully `minio/test-bucket`.
```

Подключитесь к контейнеру ```minio-0``` и проверьте, что директория bucket'а была создана в ```/data-0/data``` и ```/data-1/data```.

```bash
kubectl exec -it minio-0 -- bash

$ ls -R /data-*
/data-0:
data

/data-0/data:
test-bucket

/data-0/data/test-bucket:

/data-1:
data

/data-1/data:
test-bucket

/data-1/data/test-bucket:
```

Подключитесь к контейнеру ```minio-1``` и аналогичным образом убедитесь, что директория bucket'а также присутствует и на этом экземпляре.

## Очистка тестового окружения

```bash
kubectl delete -f minio-init.yaml -f ingress-resource.yaml -f sts.yaml -f secret.yaml
```
