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
