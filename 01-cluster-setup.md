## Топология

* DISTRIB_ID=Ubuntu
* DISTRIB_RELEASE=23.04
* DISTRIB_CODENAME=lunar
* DISTRIB_DESCRIPTION="Ubuntu 23.04"

```
+--------+ +--------+ +--------+
| kub01  | | kub02  | | kub03  |
| .151   | | .152   | | .153   |
+--------+ +--------+ +--------+
<-------192.168.133.0/24------->
```

### Конфигурация сети

```bash
sudo rm /etc/netplan/00*
sudo nano /etc/netplan/01-network.yaml
```

```yaml
network:
  ethernets:
    # interface name
    ens33:
      dhcp4: false
      # IP address/subnet mask
      addresses: [192.168.133.15x/24]
      # default gateway
      # [metric] : set priority (specify it if multiple NICs are set)
      # lower value is higher priority
      routes:
        - to: default
          via: 192.168.133.2
          metric: 100
      nameservers:
        # name server to bind
        addresses: [8.8.8.8,8.8.4.4]
        # DNS search base
      dhcp6: false
  version: 2
```
### Установка Container Runtime

В первую очередь необходимо установить следующие зависимости:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

Далее, необходимо добавить GPG-ключ репозитория Docker в менеджер пакетов (apt):

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Далее, необходимо добавить нужный репозиторий, обновить список пакетов включив в него содержимое репозитория Docker, а также установить containerd:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y containerd.io
```

После установки, проверьте успешный запуск службы containerd:

```bash
sudo service containerd status
```

Для корректной работы с Kubernetes необходимо внести несколько изменений в конфигурационный файл containerd. Сохраните конфигурацию по умолчанию containerd в файле ``/etc/containerd/config.toml``. Далее в этом файле замените значание ``SystemdCgroup = false`` на ``true`` и перезапустите сервис:

```bash
sudo containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo service containerd restart
```

Данная модификация необходима для включения полной поддержки управления systemd cgroup. Без этой опции системные контейнеры Kubernetes будут периодически перезагружаться.

### Установка Kubeadm, Kubectl и Kubelet

Следующим этапом является установка инструментов Kubernetes:

* **Kubeadm** - инструмент администрирования, работающий на уровне кластера с помощью которого можно создать новый кластер и добавлять в него дополнительные узлы.
* **Kubectl** - интерфейс командной строки, который используется для взаимодействия с кластером Kubernetes после его запуска.
* **Kubelet** - процесс Kubernetes (агент), выполняемый на рабочих узлах кластера и который отвечает за поддержание связи с плоскостью управления, а также за запуск новых контейнеров по запросу.
 
Для установки данных инструментов через менеджер пакетов (apt) зарегистрируйте GPG-ключ репозитория Google Cloud, затем добавьте репозиторий в источники и обновите список пакетов:

```bash
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --yes --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list > /dev/null
sudo apt update
```

Установите пакеты и зафиксируйте их текущие версии, чтобы менеджер пакетов не обновлял их автоматически при запуске apt upgrade. Обновление кластера Kubernetes следует инициировать вручную, чтобы избежать нежелательных изменений, вызывающих сбои и простой в работе.

```bash
sudo apt install -y kubeadm kubectl kubelet
sudo apt-mark hold kubeadm kubectl kubelet
```

### Отключение свопа

Kubernetes не работает при включенном свопе. Перед созданием кластера необходимо отключить своп. В противном случае процесс инициализации зависнет в ожидании запуска Kubelet.

Выполните эту команду, чтобы отключить своп и отключить монтирование свопа:

```bash
sudo swapoff -a
sudo sed -i 's/\/swap.img/#\/swap.img/g' /etc/fstab
```

### Загрузка модуля br_netfilter

Модуль ядра br_netfilter необходим для того, чтобы iptables мог видеть bridge-трафик. Kubeadm не позволит вам создать кластер, если этот модуль отсутствует.

Включите данный модуль и добавьте его в список загружаемых модулей при старте системы:

```bash
sudo modprobe br_netfilter
echo br_netfilter | sudo tee /etc/modules-load.d/kubernetes.conf
```

### Включение опции IP-маршрутизации

Также необходимо активировать опцию перенаправления IPv4-трафика:

```bash
sudo sudo sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/g' /etc/sysctl.conf
sudo sysctl -p
```

### Создание кластера

На виртуальной машине, выполняющей роль control-plane node, выполните команду для инициализации кластера:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Флаг ``--pod-network-cidr задает`` распределение CIDR для сетевого дополнения, которое будет установлено позже. Значение диапазона по умолчанию ``10.244.0.0/16``, однако может быть изменено, если этого требует текущая сетевая конфигурация.

Создание кластера может занять несколько минут. Информация о ходе выполнения будет отображаться в терминале. В случае успеха вы должны увидеть следующее сообщение:

```bash
Your Kubernetes control-plane has initialized successfully!
```

В сообщении также содержится информация о том, как начать использовать кластер.

### Подготовка файла Kubeconfig

Скопируйте в терминал сгенерированные команды для файла конфигурации. Установите корректные права на данный файл, чтобы утилита kubectl могла правильно прочитать его содержимое.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Установка сетевого расширения для контейнеров

Kubernetes требует наличия в кластере сетевого расширения для контейнеров, перед добавлением рабочих узлов. 

Наиболее популярными являются Calico и Flannel. В данном случае мы установим Flannel из-за простоты установки.

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### Проверка состояния кластера

Через некоторое время выполните в терминале команду ```kubectl get nodes```. Вы должны увидеть, что ваш основной узел отображается как Ready, и вы можете начать взаимодействие с кластером.

```bash
$ kubectl get nodes
NAME    STATUS   ROLES           AGE    VERSION
kub01   Ready    control-plane   9m1s   v1.27.4
```

Если выполнить команду ```kubectl get pods --all-namespaces```, то можно увидеть, что control-plane компоненты CoreDNS и Flannel запущены и работают:

```bash
$ kubectl get pods --all-namespaces
NAMESPACE      NAME                            READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-vlnl9           1/1     Running   0          39s
kube-system    coredns-5d78c9869d-ksmbz        1/1     Running   0          7m47s
kube-system    coredns-5d78c9869d-x5fch        1/1     Running   0          7m47s
kube-system    etcd-kub01                      1/1     Running   0          8m2s
kube-system    kube-apiserver-kub01            1/1     Running   0          8m
kube-system    kube-controller-manager-kub01   1/1     Running   0          8m
kube-system    kube-proxy-vmdjr                1/1     Running   0          7m47s
kube-system    kube-scheduler-kub01            1/1     Running   0          8m
```

### Добавление рабочих узлов в кластер

На каждом узле должны быть установлены containerd, Kubeadm и Kubelet. Также необходимо убедиться, что все узлы (включая control-plane) имеют полную сетевую связность между собой.

Для добавлния узлов в кластер необходимо передать команде kubeadm параметры токена и хэша сертификата с контрольного узла. 

Для этого на контрольном узле необходимо выполнить следующие команды:

```bash
$ kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION
                               EXTRA GROUPS
37akio.8r1u4nczei1gp5ia   23h         2023-07-31T12:35:38Z   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token

$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
90df233e03bfbc4666018e27027724e9568182d5af4aacc7bc7621fbc7b3a4b1
```
Далее, на узлах, которые необходимо добавить в кластер как рабочие, выполните соответствующую команду (подставьте актуальные значения токена и хэша).

```bash
sudo kubeadm join kub01:6443 --node-name [worker_node_name] --token 37akio.8r1u4nczei1gp5ia  --discovery-token-ca-cert-hash sha256:90df233e03bfbc4666018e27027724e9568182d5af4aacc7bc7621fbc7b3a4b1
```

На контрольном узле проверяем, что рабочие узлы успешно добавлены в кластер:

```bash
$ kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
kub01   Ready    control-plane   69m   v1.27.4
kub02   Ready    <none>          55m   v1.27.4
kub03   Ready    <none>          54m   v1.27.4
```

## Удаление узлов из кластера

На control-pane узле введите команду ```kubectl get nodes```, чтобы отобразить информацию о состоянии узлов в кластере.

```bash
$ kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
kub01   Ready    control-plane   25h   v1.27.4
kub02   Ready    <none>          24h   v1.27.4
kub03   Ready    <none>          24h   v1.27.4
```

Введите следующую команду, чтобы посмотреть используется ли узел, который вы хотите вывести из кластера (в данном примере kub03), для размещения pod'ов.

```bash
$ kubectl describe pods | grep Node:
Node:             kub02/192.168.133.152
Node:             kub03/192.168.133.153
Node:             kub03/192.168.133.153
Node:             kub02/192.168.133.152
Node:             kub03/192.168.133.153
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
Node:             kub03/192.168.133.153
Node:             kub02/192.168.133.152
Node:             kub03/192.168.133.153
```

На control-pane узле введите следующую команду для того чтобы вывести все активные ресурсы с удаляемого узла, а затем удалите узел из кластера.

```bash
kubectl drain kub03 --ignore-daemonsets --delete-local-data
kubectl delete node kub03
```

Проверьте, что нужное количество pod'ов размещено на свободных узлах.

```bash
$ kubectl describe pods | grep Node:
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
Node:             kub02/192.168.133.152
```

На узле, который был выведен из кластера выполните следующую команду, чтобы отчистить конфигурацию kubeadm.

```bash
sudo kubeadm reset
```
