Helm — это пакетный менеджер для Kubernetes, который упрощает процесс управления приложениями и сервисами в кластерах Kubernetes. Он позволяет упаковывать, конфигурировать и устанавливать приложения и их зависимости в Kubernetes.

Helm используется для создания предварительно настроенных пакетов приложений, называемых "чартами" (не путать с чертами). Это упрощает развертывание и управление сложными приложениями, обеспечивая консистентность и стандартизацию настроек.

Helm удобно использовать в следующих случаях:
- **Управление сложными приложениями**: когда приложение включает в себя множество компонентов Kubernetes, таких как поды, службы, тома хранения и другие ресурсы.
- **Повторное использование**: когда нужно развертывать одинаковые или похожие приложения в различных средах (разработка, тестирование, продакшн) с возможностью настройки через конфигурационные файлы.
- **Обновления и откаты** (не применимо к 44- и 223-ФЗ): для управления версиями приложений и их компонентов, облегчая процесс обновления и возможность отката к предыдущим версиям.

Часто Helm сравнивают с пакетными менеджерами в Linux:

- **Стандартизация**: так же, как пакетные менеджеры в Linux стандартизируют установку и обновление программного обеспечения, Helm стандартизирует развертывание приложений в Kubernetes.
- **Управление зависимостями**: helm автоматически разрешает зависимости приложений, подобно тому, как это делают apt или yum для пакетов.
- **Версионирование и откаты**: helm позволяет легко откатываться к предыдущим версиям чартов, аналогично механизмам отката в пакетных менеджерах.

Таким образом, использование Helm значительно упрощает процесс развертывания и управления приложениями в Kubernetes, делая его более эффективным и менее подверженным ошибкам по сравнению с традиционными методами работы с кластерами Kubernetes.

## Пакетное развертывание ресурсов с помощью Helm
### Установка Helm v3
Helm имеет сценарий установки, который автоматически получает последнюю версию Helm и устанавливает ее локально.

```bash
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

Скрипт установки хорошо документирован, поэтому перед его выполнением можно ознакомиться с ним и понять, что он делает.

Более подробная информация о других вариантах установки находится здесь: https://helm.sh/docs/intro/install/

Helm использует конфигурацию kubectl для доступа к кластеру Kubernetes. Таким образом, при установке Helm на контрольный узел кластера, где уже работает kubectl, не требуется дополнительных шагов по настройке Helm.

В случае установки Helm на машину вне кластера, перед установкой Helm необходимо произвести установку и настройку kubectl.

Структура Helm чарта
На примере сервиса sentry (http://tfs:8080/tfs/Gulfstream/DevOps/_git/sentry-helm-chart), helm чарт будет иметь следующую структуру:

```bash
.
├── Chart.yaml
├── values.yaml
├── templates
│   ├── ingress-resource.yaml
│   ├── sentry-appconfig.yaml
│   ├── sentry-deployment.yaml
│   └── sentry-service.yaml
├── sentry-0.0.1.tgz
└── .helmignore
```

**Chart.yaml**: Основной файл метаданных чарта. Он содержит информацию о чарте, такую как имя, версия и описание.

```bash
name: sentry
description: sentry service
version: 0.0.1
apiVersion: v2
keywords:
  - sentry
```

**values.yaml**: Здесь определяются значения по умолчанию для параметров чарта, которые можно переопределить при установке чарта.

Файл имеет произвольную структуру в рамках разметки YAML. В данном примере определяются значения для

- названия репозитория
- названия образа
- тега образа
- количества реплик
- DNS-корень 
- различные параметры конфигурации окружения

```bash
imageRegistry: docker-registry.gulfstream.ru
imageName: sentry
imageTag: latest

replicasCount: 1

global:
  dnsRoot: gos.services

AjaxConfig__BaseUrl: https://api.ajax.systems/api
CmaConfig__BaseUrl: http://cma-dev:3333
CNordConfig__BaseUrl: http://10.11.86.12:8200
[остальные значения опущены]
```

**templates/**: Эта папка содержит шаблоны Kubernetes, которые Helm инстанцирует при установке чарта.

**sentry-deployment.yaml**: Описывает Deployment ресурс.

В данном примере мы подставляем значения (Values) для параметров количества реплик, имени образа и порт контейнера.

Обратите внимание, что Helm использует т.н. "Go Templating Language" (больше примеров здесь: https://developer.hashicorp.com/nomad/tutorials/templates/go-template-syntax),  который, в простой нотации позволяет подставлять значения из любой произвольной YAML-структуры.

Необходимо обратить внимание, что шаблон **{{ .Release.Namespace }}** подставляет значение пространства имен в Kubernetes кластере (по умолчанию default), в котором развернут Helm чарт. 

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    service: sentry
  name: sentry
spec:
  replicas: {{ .Values.replicasCount }}
  selector:
    matchLabels:
      service: sentry
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        service: sentry
    spec:
      containers:
      - name: sentry
        image: {{ .Values.imageRegistry }}/{{ .Values.imageName }}.{{ .Release.Namespace }}:{{ .Values.imageTag }}
        ports:
        - containerPort: {{ .Values.WebConfig__Port }}
        envFrom:
        - configMapRef:
            name: sentry-appconfig
      restartPolicy: Always
```

**sentry-service.yaml**: Описывает Service для доступа к сервису.

Аналогично, подставляются значение для номера порта.

```bash
apiVersion: v1
kind: Service
metadata:
  labels:
    service: sentry
  name: sentry
spec:
  ports:
    - protocol: TCP
      port: {{ .Values.WebConfig__Port }}
  selector:
    service: sentry
```

**sentry-appconfig.yaml**: Описывает Config Map ресурс для конфигурации переменных окружения контейнера.

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: sentry-appconfig
data: 
  "AjaxConfig__ApiKey": {{ .Values.AjaxConfig__ApiKey }}
  "AjaxConfig__BaseUrl": {{ .Values.AjaxConfig__BaseUrl }}

[остальные значения опущены]
```

**ingress-resource.yaml**: Описывает Ingress ресурс для конфигурации доступа к сервису через Nginx Ingress Controller (предполагается, что он уже развернут в ходу установки и начальной конфигурации Kubernetes кластера).

```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-{{ .Values.imageName }}-{{ .Release.Namespace }}
spec:
  ingressClassName: nginx
  rules:
  - host: {{ .Values.imageName }}.{{ .Release.Namespace }}.{{ .Values.global.dnsRoot }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sentry
            port:
              number: {{ .Values.WebConfig__Port }}
```

**sentry-0.0.1.tgz**: Упакованный Helm чарт, готовый к загрузке в Helm репозиторий.

**.helmignore**: Файл, аналогичный .gitignore, указывающий на файлы и папки, которые Helm должен игнорировать при упаковке чарта.

Дополнительные файлы при необходимости могут включать README.md, LICENSE и другие документационные ресурсы.

### Установка локального Helm чарта

Чтобы установить локально созданный Helm чарт из предыдущего примера, необходимо убедиться корректности структуры чарта и в том, что Helm может подставить все необходимые значения в шаблон.

Это можно сделать с помощью команды " helm install --generate-name --dry-run --debug -n dev --create-namespace .", где

- --generate-name генерирует произвольное имя

- --dry-run симулирует установку

- --debug позволяет посмотреть финальную структуру чарта с подставленными значениями

- -n dev создает пространство имен (namespace) с именем dev

- --create-namespace создает пространство имен в кластере Kubernetes если оно не создано

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm install --generate-name --dry-run --debug -n dev --create-namespace .
install.go:214: [debug] Original chart version: ""
install.go:231: [debug] CHART PATH: /home/gorbachev.a/dev/sentry-helm-chart

NAME: chart-1700664772
LAST DEPLOYED: Wed Nov 22 17:52:52 2023
NAMESPACE: dev
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
AjaxConfig__BaseUrl: https://api.ajax.systems/api
CNordConfig__BaseUrl: http://10.11.86.12:8200
CmaConfig__BaseUrl: http://cma-dev:3333
WebConfig__Port: 3400
[часть значений опущены]
global:
  dnsRoot: gos.services
imageName: sentry
imageRegistry: docker-registry.gulfstream.ru
imageTag: latest
replicasCount: 1

HOOKS:
MANIFEST:
---
# Source: sentry/templates/sentry-appconfig.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sentry-appconfig
data:
  "AjaxConfig__BaseUrl": https://api.ajax.systems/api
  [часть значений опущены]
---
# Source: sentry/templates/sentry-service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    service: sentry
  name: sentry
spec:
  ports:
    - protocol: TCP
      port: 3400
  selector:
    service: sentry
---
# Source: sentry/templates/sentry-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    service: sentry
  name: sentry
spec:
  replicas: 1
  selector:
    matchLabels:
      service: sentry
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        service: sentry
    spec:
      containers:
      - name: sentry
        image: docker-registry.gulfstream.ru/sentry.dev:latest
        ports:
        - containerPort: 3400
        envFrom:
        - configMapRef:
            name: sentry-appconfig
      restartPolicy: Always
---
# Source: sentry/templates/ingress-resource.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-sentry-dev
spec:
  ingressClassName: nginx
  rules:
  - host: sentry.dev.gos.services
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sentry
            port:
              number: 3400
```

Таким образом мы видим, что все значения подставлены корректно и может произвести установку данного чарта в кластер.

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm install sentry -n dev --create-namespace .                 

NAME: sentry
LAST DEPLOYED: Wed Nov 22 18:01:49 2023
NAMESPACE: dev
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Мы можем проверить текущие релизы в конкретном пространстве имен, которые установлены с помощью Helm

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm list -n dev
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART       APP VERSION
sentry  dev             1               2023-11-22 18:01:49.291849315 +0300 MSK deployed        sentry-0.0.1
```

Также мы можем увидеть развернутый сервис со всеми зависимостями в самом Kubernetes кластере

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ kubectl get all -n dev
NAME                          READY   STATUS    RESTARTS   AGE
pod/sentry-7bc6fc8f86-lx8nn   1/1     Running   0          56s

NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/sentry   ClusterIP   10.109.62.72   <none>        3400/TCP   56s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sentry   1/1     1            1           56s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/sentry-7bc6fc8f86   1         1         1       56s
```

Проверить доступ к сервису через Ingress-контроллер можно перейдя по адресу http://sentry.dev.gos.services/api (предварительно убедившись, что корень DNS указывает на адрес Ingress-контроллера) и проверить логи контейнера.

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ kubectl logs sentry-7bc6fc8f86-lx8nn -n dev| grep "HTTP/1.1 GET"2023-11-22 18:20:02 INF Microsoft.AspNetCore.Hosting.Diagnostics Request starting HTTP/1.1 GET http://sentry.dev.gos.services/api
```

### Удаление локального Helm чарта
Удалить локальный Helm чарт можно с помощью команды helm uninstall

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm uninstall sentry -n dev
release "sentry" uninstalled
```

## Работа с Helm репозиторием
После создания Helm чарта локально, его можно добавить в т.н. Helm репозиторий для последующего версионирования и включения версий данного чарта в зависимости других чартов.

В данном примере в качестве Helm репозитория используется Nexus OSS (https://nexus.gulfstream.ru/#browse/browse:helm). Альтернативными сервисами для организации Helm репозитория могут быть такие сервисы как Amazon S3, Google Cloud Storage, GitHub Pages и т.д.

### Пакетирование Helm чарта

Перед добавлением Helm чарта в репозиотрий, необходимо его запаковать в архив tgz следующей командой ```helm package .```.

На примере Helm чарта sentry будет создан файл sentry-0.0.1.tgz, на основе мета-информации в файле Chart.yaml.

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ ll *tgz
-rw-rw-r-- 1 gorbachev.a gorbachev.a 56578 Nov 17 12:43 sentry-0.0.1.tgz
```

### Загрузка чарта в Helm репозиторий
Helm репозиторий, по сути, представляет собой объектное хранилище, доступ к которому осуществляется по протоколу HTTP. Таким образом, загрузку можно осуществить с помощью утилиты curl, используя следующую команду:

```bash
curl -u user:password  https://nexus.gulfstream.ru/repository/helm/ --upload-file sentry-0.0.1.tgz
```

После этого, необходимо перестроить индекс самого репозитория, чтобы мета-данные включали в себя информацию о вновь загруженном Helm чарте.

![image](https://github.com/menus12/k8s/assets/24583341/089a4337-4d22-4a1c-a070-281a35354a8d)

Также в веб-консоли Nexus OSS можно убедиться в том, что Helm чарт нужной версии загружен в репозиторий и доступен для скачивания.

![image](https://github.com/menus12/k8s/assets/24583341/a90a8fc9-440b-4afd-960c-f5459c660b7c)

### Добавление Helm репозиотрия в локальное окружение

Добавление репозитория в локальное окружение Helm позволяет управлять доступными чартами. 

Добавить репозиторий можно с помощью команды helm repo add, например:

```bash
helm repo add gulfstream https://nexus.gulfstream.ru/repository/helm/ --username user --password password
```

Обновить информацию о чартах в локальном Helm, выполнив команду helm repo update:

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "gulfstream" chart repository
Update Complete. ⎈Happy Helming!⎈
```

Посмотреть список добавленных репозиториев можно с помощью команды helm repo list:

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm repo list
NAME            URL
gulfstream      https://nexus.gulfstream.ru/repository/helm/
```

### Установка чартов из Helm-репозитория

Посмотреть список доступных helm чартов можно с помощью команды helm search:

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm search repo gulfstream
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
gulfstream/gulfstream-root      0.0.1           0.0.1           Root helm chart of Gulfstream services
gulfstream/sentry               0.0.1                           sentry service
gulfstream/videoanalytics       0.0.1                           videoanalytics service
```

Установить Helm чарт из репозитория (на примере сервиса sentry) можно с помощью команд helm install и helm upgrade. Вторая команда позволяет также обновлять ревизию чарта и устанавливать его, если сервис еще не установлен:

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm upgrade --install sentry gulfstream/sentry --namespace dev --create-namespace
Release "sentry" does not exist. Installing it now.
NAME: sentry
LAST DEPLOYED: Mon Nov 27 17:33:03 2023
NAMESPACE: dev
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Проверить текущую ревизию установленных через helm чартов можно с помощью команды helm list:

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm list -n dev
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART       APP VERSION
sentry  dev             1               2023-11-27 17:33:03.037454486 +0300 MSK deployed        sentry-0.0.1
```

Также можно проверить, что сервис установлен и работает в консоли kubectl:

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ kubectl get all -n dev
NAME                          READY   STATUS    RESTARTS   AGE
pod/sentry-7bc6fc8f86-44q6t   1/1     Running   0          35s

NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/sentry   ClusterIP   10.108.254.129   <none>        3400/TCP   35s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sentry   1/1     1            1           35s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/sentry-7bc6fc8f86   1         1         1       35s
```

Удалить установленный Helm чарт можно командой helm uninstall:

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ helm uninstall sentry -n dev
release "sentry" uninstalled
```

Обратите внимание на то, что пространство имен (dev) не удаляется вместе с чартом и остается в конфигурации Kubernetes. Его необходимо удалять отдельно через kubectl:

```bash
gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ kubectl get ns
NAME              STATUS   AGE
argocd            Active   26d
cert-manager      Active   28d
default           Active   33d
dev               Active   9d
ingress-nginx     Active   33d
kube-flannel      Active   33d
kube-node-lease   Active   33d
kube-public       Active   33d
kube-system       Active   33d
metallb-system    Active   33d

gorbachev.a@k8s-d1:~/dev/sentry-helm-chart$ kubectl delete ns dev
namespace "dev" deleted
```

## Создание корневого Helm чарта

Helm, как аналог пакетного менеджера в Linux, имеет возможность определять зависимости в мета-данных, которые могу ссылаться на другие Helm чарты определенных версий из определенных репозиториев.

### Структура корневого Helm чарта

Структура чарта, который не содержит никаких ресурсов для кластера Kubernetes, но который должен ссылаться на другие Helm чарты, довольно простая:

```bash
.
├── Chart.yaml
└── values.yaml
```

**Chart.yaml** также содержит мета-данные, такие как название, версия и описание чарта, а также дополнительно содержит информацию о зависимостях.

На примере корневого Helm чарта gulfstream-root (http://tfs:8080/tfs/Gulfstream/DevOps/_git/gulfstream-root-helm-chart), который ссылается на существующие чарты сервисов sentry (http://tfs:8080/tfs/Gulfstream/DevOps/_git/sentry-helm-chart) и videoanalytics (http://tfs:8080/tfs/Gulfstream/DevOps/_git/videoanalytics-helm-chart):

```bash
apiVersion: v2
name: gulfstream-root
description: Root helm chart of Gulfstream services
type: application
version: 0.0.1
appVersion: "0.0.1"
dependencies:
- name: sentry
  version: "0.0.1"
  repository: "https://nexus.gulfstream.ru/repository/helm/"
- name: videoanalytics
  version: "0.0.1"
  repository: "https://nexus.gulfstream.ru/repository/helm/"
```

**values.yaml** является опциональным и служит для того, чтобы переопределить общие значения (переменные) в субчартах:

```bash
global:
  dnsRoot: gos.services
  Vault__Url: https://vault-dev.gl.gulfstream.ru:8200
  Vault__Login: admin
  Vault__Password: P@ssw0rd
sentry:
  WebConfig__Port: 3400
videoanalytics:
  WebConfig__Port: 15111
```

Обратите внимание на то, что для переопределения переменных в конкретном субчарте необходимо это делать в словаре соответствующем YAML-структуре. В примере выше оба сервиса используют переменную WebConfig__Port, которая имеет разные значения. Если требуется назначить значение переменной для всех субчартов, необходимо чтобы в самих субчартах данные переменные были в словаре global, как в примере выше.

### Установка зависимостей

Однако при попытке развернуть данный Helm чарт, Helm выдаст ошибку отсутствия зависимостей:

```bash
gorbachev.a@k8s-d1:~/dev/gulfstream-root-helm-chart$ helm install gulfstream-root . -n dev --create-namespace
Error: INSTALLATION FAILED: An error occurred while checking for chart dependencies. You may need to run `helm dependency build` to fetch missing dependencies: found in Chart.yaml, but missing in charts/ directory: sentry, videoanalytics
```

Подтянуть зависимости можно с помощью команды helm dependency build:

```bash
gorbachev.a@k8s-d1:~/dev/gulfstream-root-helm-chart$ helm dependency build
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "gulfstream" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 2 charts
Downloading sentry from repo https://nexus.gulfstream.ru/repository/helm/
Downloading videoanalytics from repo https://nexus.gulfstream.ru/repository/helm/
Deleting outdated charts

gorbachev.a@k8s-d1:~/dev/gulfstream-root-helm-chart$ tree charts/
charts/
├── sentry-0.0.1.tgz
└── videoanalytics-0.0.1.tgz
```

### Установка корневого Helm чарта

Далее, все шаги по упаковке и добавлению чарта в репозиторий аналогичны описанным выше:

```bash
gorbachev.a@k8s-d1:~/dev/gulfstream-root-helm-chart$ helm package .
Successfully packaged chart and saved it to: /home/gorbachev.a/dev/gulfstream-root-helm-chart/gulfstream-root-0.0.1.tgz

gorbachev.a@k8s-d1:~/dev/gulfstream-root-helm-chart$ curl -u user:password  https://nexus.gulfstream.ru/repository/helm/ --upload-file gulfstream-root-0.0.1.tgz
```

Далее, шаги по установке и проверки работы данного чарта также аналогичны шагам, описанным выше:

```bash
gorbachev.a@k8s-d1:~/dev/gulfstream-root-helm-chart$ helm upgrade --install gulfstream-root gulfstream/gulfstream-root --namespace dev --create-namespace
Release "gulfstream-root" does not exist. Installing it now.
NAME: gulfstream-root
LAST DEPLOYED: Mon Nov 27 20:25:08 2023
NAMESPACE: dev
STATUS: deployed
REVISION: 1
TEST SUITE: None

gorbachev.a@k8s-d1:~/dev/gulfstream-root-helm-chart$ helm list -n dev
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
gulfstream-root dev             1               2023-11-27 20:25:08.326397912 +0300 MSK deployed        gulfstream-root-0.0.1   0.0.1

gorbachev.a@k8s-d1:~/dev/gulfstream-root-helm-chart$ kubectl get all -n dev
NAME                                  READY   STATUS    RESTARTS   AGE
pod/sentry-7bc6fc8f86-6ptw7           1/1     Running   0          26s
pod/videoanalytics-59f8575ccb-tvxfg   1/1     Running   0          26s

NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
service/sentry           ClusterIP   10.104.63.232    <none>        3400/TCP    26s
service/videoanalytics   ClusterIP   10.102.235.219   <none>        15111/TCP   26s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sentry           1/1     1            1           26s
deployment.apps/videoanalytics   1/1     1            1           26s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/sentry-7bc6fc8f86           1         1         1       26s
replicaset.apps/videoanalytics-59f8575ccb   1         1         1       26s
```

Удаление чарта выполняется также с помощью команды helm uninstall:

```bash
gorbachev.a@k8s-d1:~/dev/gulfstream-root-helm-chart$ helm uninstall gulfstream-root -n dev
release "gulfstream-root" uninstalled
```
