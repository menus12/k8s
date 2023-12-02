ArgoCD - это декларативное, GitOps-ориентированное средство управления развертыванием ресурсов для кластеров Kubernetes.

## Установка ArgoCD 

### Развертывание ArgoCD в кластере Kubernetes

Развернуть ArgoCD в существующем кластере можно несколькими способами - через YAML-манифест Kubernetes и с помощью Helm. Следуя инструкциям на официальном сайте, в данном примере, мы ограничимся установкой через простой YAML-манифест, для которого также необходимо создать отдельное пространство имен:

```bash
gorbachev.a@k8s-d1:~/dev/k8s$ kubectl create namespace argocd
namespace/argocd created

gorbachev.a@k8s-d1:~/dev/k8s$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

gorbachev.a@k8s-d1:~/dev/k8s$ kubectl get all -n argocd
NAME                                                    READY   STATUS    RESTARTS   AGE
pod/argocd-application-controller-0                     1/1     Running   0          5m50s
pod/argocd-applicationset-controller-5bf97c679b-xhq27   1/1     Running   0          5m50s
pod/argocd-dex-server-f7648d898-9dqmq                   1/1     Running   0          5m50s
pod/argocd-notifications-controller-6cf7579685-8jcq6    1/1     Running   0          5m50s
pod/argocd-redis-6976fc7dfc-n5bhq                       1/1     Running   0          5m50s
pod/argocd-repo-server-8477fdffc7-fnzl4                 1/1     Running   0          5m50s
pod/argocd-server-7c7d77f474-45q2j                      1/1     Running   0          5m50s

NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/argocd-applicationset-controller          ClusterIP   10.104.219.16    <none>        7000/TCP,8080/TCP            5m50s
service/argocd-dex-server                         ClusterIP   10.109.32.120    <none>        5556/TCP,5557/TCP,5558/TCP   5m50s
service/argocd-metrics                            ClusterIP   10.106.232.203   <none>        8082/TCP                     5m50s
service/argocd-notifications-controller-metrics   ClusterIP   10.97.238.100    <none>        9001/TCP                     5m50s
service/argocd-redis                              ClusterIP   10.108.33.225    <none>        6379/TCP                     5m50s
service/argocd-repo-server                        ClusterIP   10.97.13.173     <none>        8081/TCP,8084/TCP            5m50s
service/argocd-server                             ClusterIP   10.109.32.64     <none>        80/TCP,443/TCP               5m50s
service/argocd-server-metrics                     ClusterIP   10.99.86.242     <none>        8083/TCP                     5m50s

NAME                                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/argocd-applicationset-controller   1/1     1            1           5m50s
deployment.apps/argocd-dex-server                  1/1     1            1           5m50s
deployment.apps/argocd-notifications-controller    1/1     1            1           5m50s
deployment.apps/argocd-redis                       1/1     1            1           5m50s
deployment.apps/argocd-repo-server                 1/1     1            1           5m50s
deployment.apps/argocd-server                      1/1     1            1           5m50s

NAME                                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/argocd-applicationset-controller-5bf97c679b   1         1         1       5m50s
replicaset.apps/argocd-dex-server-f7648d898                   1         1         1       5m50s
replicaset.apps/argocd-notifications-controller-6cf7579685    1         1         1       5m50s
replicaset.apps/argocd-redis-6976fc7dfc                       1         1         1       5m50s
replicaset.apps/argocd-repo-server-8477fdffc7                 1         1         1       5m50s
replicaset.apps/argocd-server-7c7d77f474                      1         1         1       5m50s

NAME                                             READY   AGE
statefulset.apps/argocd-application-controller   1/1     5m50s
```

По умолчанию API-сервер Argo CD имеет IP-адрес только для доступа внутри кластера . Чтобы получить доступ к API-серверу из вне, необходимо выбрать способ публикации доступа:

- Через проброс портов (Port Forwarding)
- Через Ingress-контроллер
- Через балансировщик нагрузки

В данном примере мы производим установку в кластер где уже установлены и работают Ingress-контроллер (Nginx) и балансировщик нагрузки (Metallb), однако, предполагется, что установленный Ingress-контроллер выполняет роль предоставления доступа непосредственно к приложениям (продуктовой нагрузке), поэтому в данном примере мы будем предоставлять внешний доступ к ArgoCD через балансировщик нагрузки.

```bash
gorbachev.a@k8s-d1:~/dev/k8s$ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
service/argocd-server patched
gorbachev.a@k8s-d1:~/dev/k8s$ kubectl get svc -n argocd
NAME                                      TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP      10.104.219.16    <none>         7000/TCP,8080/TCP            7m36s
argocd-dex-server                         ClusterIP      10.109.32.120    <none>         5556/TCP,5557/TCP,5558/TCP   7m36s
argocd-metrics                            ClusterIP      10.106.232.203   <none>         8082/TCP                     7m36s
argocd-notifications-controller-metrics   ClusterIP      10.97.238.100    <none>         9001/TCP                     7m36s
argocd-redis                              ClusterIP      10.108.33.225    <none>         6379/TCP                     7m36s
argocd-repo-server                        ClusterIP      10.97.13.173     <none>         8081/TCP,8084/TCP            7m36s
argocd-server                             LoadBalancer   10.109.32.64     10.11.100.82   80:31798/TCP,443:31754/TCP   7m36s
argocd-server-metrics                     ClusterIP      10.99.86.242     <none>         8083/TCP                     7m36s
```

После внесения изменений в конфигурацию мы видим, что у сервиса argocd-server появился внешний IP-адрес (EXTERNAL-IP) 10.11.100.82 из пула, присвоенного балансировщику нагрузки.

Проверка доступа показывает, что веб-интерфейс ArgoCD доступен по данному адресу.

![image](https://github.com/menus12/k8s/assets/24583341/2f0fba51-1681-4c3f-88a9-d01aa4f1f646)

### Получение доступа к веб-интерфейсу
По умолчанию ArgoCD создает учетную запись администратора admin. Пароль можно извлечь из ресурса secret, который был создан при установке:

```bash
gorbachev.a@k8s-d1:~/dev/k8s$ kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d; echo
6q-B-zobIhck0r1X
```

При входе в веб-интерфейс с данным паролем его необходимо сменить:

![image](https://github.com/menus12/k8s/assets/24583341/4cac0d3d-3def-46c1-9c76-6f70ae8e5fd8)

## Работа с ArgoCD

### Добавление Helm репозитория

Для начала работы с ArgoCD необходимо добавить в его настройках кластер Kubernetes, куда будет производиться установка, а также репозиторий (Git или Helm), если установка будет производиться из приватных репозиториев.

При установке ArgoCD в кластер Kubernetes, ArgoCD автоматически добавляет данный кластер в свои настройки, дополнительных действий не требуется.

Для того чтобы добавить приватный Helm репозиторий, необходимо перейти в Settings → Repositories → + Connect Repo и добавить соответствующий репозиторий:

![image](https://github.com/menus12/k8s/assets/24583341/34889c39-68b9-4efc-afbc-3834b835c077)

После добавления статус подключения репозитория должен отображаться как Successful:

![image](https://github.com/menus12/k8s/assets/24583341/87896b1f-dc2d-4cb2-9355-265b3f7f4704)

### Установка приложения

Для установки приложений необходимо перейти в раздел Application → + New App и заполнить несколько разделов.

В разделе General необходимо выбрать политику синхронизации:

- Manual - синхронизация приложения с источником будет производиться вручную - рекомендуется для продуктовой среды
- Automatic - автоматическая (декларативная) синхронизация с источником - может использоваться для среды разработки и тестирования

![image](https://github.com/menus12/k8s/assets/24583341/b1266e33-1ae2-43f5-a5d9-265a074b7d37)

При выборе автоматической синхронизации также можно выбрать опции:

- Prune resources - ArgoCD будет автоматически удалять ресурсы, которые больше не определены в источнике синхронизации. 
- Self heal - ArgoCD будет приводить развертывание к декларативному состоянию, которое описано в источнике при нахождении девиаций - например, если какие-либо изменения в кластере были внесены в ручную, они будут удаляться при синхронизации с источником.
- Auto-create namespace - автоматическое создание пространство имен в кластере Kubernetes, если оно не было создано ранее.

В разделе Source необходимо выбрать ранее добавленный репозиторий (или вставить адрес публичного репозитория) и чарт (в случае Helm, манифест - в случае Git), а также версию чарта.

В разделе Destination необходимо выбрать кластер (по умолчанию), а также пространство имен, куда будет производиться развертывание.

![image](https://github.com/menus12/k8s/assets/24583341/6ec4cc41-2379-4435-857d-fdd36a8150f7)

В случае развертывания Helm чарта, в разделе Helm можно посмотреть доступные значения (Values), а также переопределить данные значения:

![image](https://github.com/menus12/k8s/assets/24583341/2d1a0a6b-da1f-46b4-afbd-cc0c69f0c92e)

После определения всех параметров, для развертывания приложения нужно нажать на CREATE в верху страницы и далее можно перейти на экран приложения, чтобы увидеть статус синхронизации:

![image](https://github.com/menus12/k8s/assets/24583341/16bd8a4d-c246-405f-9b9f-53f84e47ceb6)

### Создание пользователей и разграничение прав доступа

В рамках ArgoCD управлять доступом можно на уровне различных ресурсов:

- clusters
- projects
- applications
- repositories
- certificates
- accounts
- gpgkeys

К данным типам ресурсов можно предоставлять следующие права доступа:

- get
- create
- update
- delete
- sync
- override
- action

Для пользователей и групп пользователей, которые должны иметь возможность разворачивать приложения без доступа к администрированию самого ArgoCD, удобно предоставлять права на уровне приложений (applications) в рамках тех или иных проектов (projects). В данном примере мы создадим проекты dev и prod, а также двух пользователей junior и senior. Пользователю senior будет доступно управление всеми приложениями во всех пространствах имен кластера в обоих проектах, а пользователю junior - только управлениями приложениями в проекте dev, в пространстве имен dev кластера Kubernetes.

Для того, чтобы создать пользователей, необходимо отредактировать ресурс Config Map argocd-cm:

```bash
gorbachev.a@k8s-d1:~$ kubectl edit cm argocd-cm -n argocd
```

Редактирование Config Map откроется в текстовом редакторе (по умолчанию vi/vim):

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"labels":{"app.kubernetes.io/name":"argocd-cm","app.kubernetes.io/part-of":"argocd"},"name":"argocd-cm","namespace":"argocd"}}
  creationTimestamp: "2023-11-28T11:39:30Z"
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-cm
  namespace: argocd
  resourceVersion: "5509481"
  uid: cb2fb5b8-db08-4ac0-bba2-c8a7ac92be78
data:                           # словарь data необходимо 
  accounts.junior: apiKey,login # добавить в произвольном месте
  accounts.senior: apiKey,login # с соблюдением отступов разметки YAML
```

Необходимо сохранить Config Map и выйти из редактора (команда для vim: esc →  wq!). Далее необходимо задать пароль для созданных пользователей - данную операцию можно осуществить только с помощью Argo CLI, который можно установить отдельно, или использовать внутри контейнера argocd-server:

```bash
gorbachev.a@k8s-d1:~/dev/k8s$ kubectl get pod -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          2d6h
argocd-applicationset-controller-5bf97c679b-xhq27   1/1     Running   0          2d6h
argocd-dex-server-f7648d898-9dqmq                   1/1     Running   0          2d6h
argocd-notifications-controller-6cf7579685-8jcq6    1/1     Running   0          2d6h
argocd-redis-6976fc7dfc-n5bhq                       1/1     Running   0          2d6h
argocd-repo-server-8477fdffc7-fnzl4                 1/1     Running   0          2d6h
argocd-server-7c7d77f474-45q2j                      1/1     Running   0          2d6h # <-- необходимо зайти в данный контейнер

gorbachev.a@k8s-d1:~/dev/k8s$ kubectl exec -it -n argocd argocd-server-7c7d77f474-45q2j bash
argocd@argocd-server-7c7d77f474-45q2j:~$
```

Далее необходимо войти в Argo CLI:

```bash
argocd@argocd-server-7c7d77f474-45q2j:~$ argocd login --username admin --password P@ssw0rd argocd-server
WARNING: server certificate had error: tls: failed to verify certificate: x509: certificate signed by unknown authority. Proceed insecurely (y/n)? y
'admin:login' logged in successfully
Context 'argocd-server' updated
```

Убедимся в том, что пользователи созданы:

```bash
argocd@argocd-server-7c7d77f474-45q2j:~$ argocd account list
NAME    ENABLED  CAPABILITIES
admin   true     login
junior  true     apiKey, login
senior  true     apiKey, login
```

Далее зададим пароль для созданных пользователей:

```bash
argocd@argocd-server-7c7d77f474-45q2j:~$ argocd account update-password --account senior --new-password P@ssw0rd
*** Enter password of currently logged in user (admin):
Password updated

argocd@argocd-server-7c7d77f474-45q2j:~$ argocd account update-password --account junior --new-password P@ssw0rd
*** Enter password of currently logged in user (admin):
Password updated
```

Далее можно выйти из контейнера:

```bash
argocd@argocd-server-7c7d77f474-45q2j:~$ exit
gorbachev.a@k8s-d1:~/dev/k8s$
```

Далее, необходимо настроить проекты prod и dev в веб-консоли ArgoCD в разделе Settings → Projects → + New Project. 

В настройках проекта prod необходимо добавить репозиторий источника (по умолчанию *), а также кластер назначения и пространство имен внутри кластера:

![image](https://github.com/menus12/k8s/assets/24583341/fb0484f0-7c5c-4e79-a6be-b59ed3d9059f)

В настройках проекта dev необходимо настроить те же параметры, ограничив пространство имен внутри кластера:

![image](https://github.com/menus12/k8s/assets/24583341/de16af96-46ec-460c-bb17-5099ae01a08f)

Далее, необходимо внести изменения в Config Map argocd-rbac-cm для определения ролей и прав доступа:

```bash
gorbachev.a@k8s-d1:~$ kubectl edit cm argocd-rbac-cm -n argocd
```

Редактирование Config Map откроется в текстовом редакторе (по умолчанию vi/vim):

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ConfigMap","metadata":{"annotations":{},"labels":{"app.kubernetes.io/name":"argocd-rbac-cm","app.kubernetes.io/part-of":"argocd"},"name":"argocd-rbac-cm","namespace":"argocd"}}
  creationTimestamp: "2023-11-28T11:39:30Z"
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-rbac-cm
  namespace: argocd
  resourceVersion: "5161027"
  uid: 50e9ed84-ae46-4ff9-904b-e96354e90516
data:                                               # словарь data необходимо добавить в произвольном месте с соблюдением отступов разметки YAML
  policy.default: role:readonly                     # политика по умолчанию - все ресурсы будут доступны только для чтения
  policy.csv: |                                     # далее политика доступа определяется следующим образом
    p, role:senior, applications, *, */*, allow     # роли senior разрешено (allow) проводить любые операции (*) на уровне приложений (applications) с любыми ресурсами внутри всех проектов (*/*)
    p, role:junior, applications, *, dev/*, allow   # роли junior разрешено (allow) проводить любые операции (*) на уровне приложений (applications) с любыми ресурсами внутри проекта dev (dev/*)
    g, junior, role:junior                          # пользователю junior назначается роль junior
    g, senior, role:senior                          # пользователю senior назначается роль senior
```

Необходимо сохранить Config Map и выйти из редактора (команда для vim: esc →  wq!). 

Далее можно проверить работу разграничения прав доступа для пользователя junior. При попытке развернуть Helm чарт в проекте prod интерфейс покажет ошибку:

![image](https://github.com/menus12/k8s/assets/24583341/c6c3b12c-c385-486c-8a27-abaf05305e53)

Также при попытке развернуть Helm чарт в проекте dev, но в каком-либо другом пространстве имен внутри кластера интерфейс также выдаст ошибку:

![image](https://github.com/menus12/k8s/assets/24583341/22c09e79-ec17-4425-a1f7-f92a02a83a34)

При попытке развернуть Helm чарт в проекте dev в пространстве имен dev, приложение успешно развернется:

![image](https://github.com/menus12/k8s/assets/24583341/c5a0b7f4-3476-410c-867f-314d7077a641)

Аналогично, пользователь senior может развернуть Helm чарт в проекте prod в пространстве имен dev:

![image](https://github.com/menus12/k8s/assets/24583341/ce708f91-e1f2-4c48-a770-05eae8285994)



