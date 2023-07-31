## Получение информации о состоянии узов в кластере

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

## Вывод узла из кластера

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

## Сброс параметров kubeadm

На узле, который был выведен из кластера выполните следующую команду, чтобы отчистить конфигурацию kubeadm.

```bash
sudo kubeadm reset
```
