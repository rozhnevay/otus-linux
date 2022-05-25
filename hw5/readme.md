# **Homework 5**
Переключимся на суперпользователя
```
    sudo su
```

### **Настройка сервера NFS**
Доустанавливаем
```bash
    yum install nfs-utils
```
Настраиваем firewall
```bash
    systemctl enable firewalld --now
    systemctl status firewalld
    firewall-cmd --add-service="nfs3" \
    --add-service="rpc-bind" \
    --add-service="mountd" \
    --permanent
    firewall-cmd --reload
```
Включаем сервис NFS и проверяем порты
```
    systemctl enable nfs --now
    ss -tnplu
```
Убедились, что нужные порты появились. 
Создаем директорию для экспорта
```
    mkdir -p /srv/share/upload
    chown -R nfsnobody:nfsnobody /srv/share
    chmod 0777 /srv/share/upload
```
Добавляем директорию в exports и экспортируем
```
    cat << EOF > /etc/exports
    /srv/share 192.168.50.11/32(rw,sync,root_squash)
    EOF
    exportfs -r
    exportfs -s
```
```
    /srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```
### **Настройка клиента NFS**
Устанавливаем утилиту + включаем файервол
```
    yum install nfs-utils
    systemctl enable firewalld --now
    systemctl status firewalld
```
Добавляем в fstab опцию для автомонтирования и перегружаем демона
```
    echo "192.168.50.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
    systemctl daemon-reload
    systemctl restart remote-fs.target
```
Заходим в /mnt и проверяем успешность монтирования
```
    cd /mnt
    mount | grep mnt
```
```
    systemd-1 on /mnt type autofs (rw,relatime,fd=44,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=17616)
    192.168.50.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport
    =20048,mountproto=udp,local_lock=none,addr=192.168.50.10)
```
Проверка работоспособности - создали файл на сервере. Видим файл на клиенте:
```
    drwxrwxrwx. 2 nfsnobody nfsnobody 24 May 25 11:49 upload
```
Проверяем сервер
```
    systemctl status nfs
```
```
    [root@nfss vagrant]# systemctl status nfs
    ● nfs-server.service - NFS server and services
       Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled; vendor preset: disabled)
       Active: active (exited) since Wed 2022-05-25 11:26:07 UTC; 25min ago
      Process: 22222 ExecStartPost=/bin/sh -c if systemctl -q is-active gssproxy; then systemctl reload gssproxy ; fi (code=exited, status=0/SUCCESS)
      Process: 22206 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
      Process: 22205 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
     Main PID: 22206 (code=exited, status=0/SUCCESS)
       CGroup: /system.slice/nfs-server.service
    
    May 25 11:26:07 nfss systemd[1]: Starting NFS server and services...
    May 25 11:26:07 nfss systemd[1]: Started NFS server and services.
```
Проверяем firewall
```
    systemctl status firewalld
```
```
    [root@nfss vagrant]# systemctl status firewalld
    ● firewalld.service - firewalld - dynamic firewall daemon
       Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
       Active: active (running) since Wed 2022-05-25 10:47:35 UTC; 1h 4min ago
         Docs: man:firewalld(1)
     Main PID: 3312 (firewalld)
       CGroup: /system.slice/firewalld.service
               └─3312 /usr/bin/python2 -Es /usr/sbin/firewalld --nofork --nopid
    
    May 25 10:47:34 nfss systemd[1]: Starting firewalld - dynamic firewall daemon...
    May 25 10:47:35 nfss systemd[1]: Started firewalld - dynamic firewall daemon.
    May 25 10:47:35 nfss firewalld[3312]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It will be removed in a future release. ...ling it now.
    May 25 10:48:02 nfss firewalld[3312]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It will be removed in a future release. ...ling it now.
    Hint: Some lines were ellipsized, use -l to show in full.
```
Проверяем эксопрты
```
    exportfs -s
```
```
    /srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
```
Проверяем работу RPC
```
    showmount -a 192.168.50.10
```
```
    All mount points on 192.168.50.10:
    192.168.50.11:/srv/share
```
Проверяем клиента
```
    showmount -a 192.168.50.10
```
```
    All mount points on 192.168.50.10:
    192.168.50.11:/srv/share
```
```
    [vagrant@nfsc upload]$ mount | grep mnt
    systemd-1 on /mnt type autofs (rw,relatime,fd=44,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=17616)
    192.168.50.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport
    =20048,mountproto=udp,local_lock=none,addr=192.168.50.10)
```
```
    [vagrant@nfsc upload]$ cd /mnt
    [vagrant@nfsc mnt]$ ll
    total 0
    drwxrwxrwx. 2 nfsnobody nfsnobody 24 May 25 11:49 upload
    [vagrant@nfsc mnt]$
```
Пытаемся создать файл с клиента и нарываемся на Permission denied. Идем на сервер и докидываем прав для others
```
    [root@nfss share]# cd /srv
    [root@nfss srv]# ll
    total 0
    drwxr-xr-x. 3 nfsnobody nfsnobody 20 May 25 11:27 share
    [root@nfss srv]# chmod o+w share
```
Снова пытаемся создать с клиента - теперь все ок
```
    [root@nfsc mnt]# touch final_check
    [root@nfsc mnt]# ll
    total 0
    -rw-r--r--. 1 nfsnobody nfsnobody  0 May 25 12:06 final_check
    drwxrwxrwx. 2 nfsnobody nfsnobody 24 May 25 11:49 upload
```
Создание с помощью скрипта + все проверки - все ок, файл, созданный на сервере виден на клиенте и наоборот

