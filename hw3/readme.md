# **Homework 3**
Переключимся на суперпользователя
```
    sudo su
```
Установим xfsdump
```
    yum install xfsdump
```

### **Уменьшить том под / до 8G**
Смотрим какие у нас есть LV
```
    lvs
```
```
    LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
    LogVol00 VolGroup00 -wi-ao---- <37.47g
    LogVol01 VolGroup00 -wi-ao----   1.50g
```
Посмотрим, куда примонтирован "/":
```
    mount
```
```
    ...
    /dev/mapper/VolGroup00-LogVol00 on / type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
    ...
```
Поскольку это XFS, то ее уменьшить нельзя, необходимо пересоздавать VG
Подготовим временный раздел для /
```
    pvcreate /dev/sdb
    vgcreate vg_root /dev/sdb
    lvcreate -n lv_root -l +100%FREE /dev/vg_root
```
Создаем на подготовленном разделе файловую систему, смонтируем в /mnt и копируем туда содержимое /
```
    mkfs.xfs /dev/vg_root/lv_root
    mount /dev/vg_root/lv_root /mnt
    xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
```
Убедимся, что в /mnt что то есть:
```
    cd /mnt
    ls
```
```
    bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  vagrant  var
```
Переконфигурирем grub для того, чтобы при старте зайти в новый /
```
    for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
    chroot /mnt/
    grub2-mkconfig -o /boot/grub2/grub.cfg
```
Обновим образ initrd
```
    cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
    s/.img//g"` --force; done
```
В /boot/grub2/grub.cfg заменим LV rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root
```
    sed -i "s/rd.lvm.lv=VolGroup00\/LogVol00/rd.lvm.lv=vg_root\/lv_root/g" /boot/grub2/grub.cfg
```
ВЫходим из режима chroot, перезагружаем и проверяем root том
```
    exit
    reboot
    lsblk
```
```
    NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda                       8:0    0   40G  0 disk
    ├─sda1                    8:1    0    1M  0 part
    ├─sda2                    8:2    0    1G  0 part /boot
    └─sda3                    8:3    0   39G  0 part
      ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
      └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm
    sdb                       8:16   0   10G  0 disk
    └─vg_root-lv_root       253:0    0   10G  0 lvm  /
    sdc                       8:32   0    2G  0 disk
    sdd                       8:48   0    1G  0 disk
    sde                       8:64   0    1G  0 disk
```
Удаляем старый LV на 40G и создаем новый на 8G
```
    lvremove /dev/VolGroup00/LogVol00
    lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
```
Проделываем те же операции, что и в первый раз, за исключением правки grab.cfg
### **Перенести /var**
На свободных дисках создаем том для var с зеркалом: 
```
    pvcreate /dev/sdc /dev/sdd
    vgcreate vg_var /dev/sdc /dev/sdd
    lvcreate -L 950M -m1 -n lv_var vg_var
```
```
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
```
Создаем на новом LV файловую систему и перемещаем туда var
```
    mkfs.ext4 /dev/vg_var/lv_var
    mount /dev/vg_var/lv_var /mnt
    cp -aR /var/* /mnt/
```
Размонтируем старый var и примонтируем новый
```
    umount /mnt
    mount /dev/vg_var/lv_var /var
```
Правим fstab для автоматического монтирования /var:
```
     echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
```
Перезагружаемся, проверяем тома с помощью lsblk
```
    NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda                        8:0    0   40G  0 disk
    ├─sda1                     8:1    0    1M  0 part
    ├─sda2                     8:2    0    1G  0 part /boot
    └─sda3                     8:3    0   39G  0 part
      ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
      └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
    sdb                        8:16   0   10G  0 disk
    └─vg_root-lv_root        253:7    0   10G  0 lvm
    sdc                        8:32   0    2G  0 disk
    ├─vg_var-lv_var_rmeta_0  253:2    0    4M  0 lvm
    │ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
    └─vg_var-lv_var_rimage_0 253:3    0  952M  0 lvm
      └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
    sdd                        8:48   0    1G  0 disk
    ├─vg_var-lv_var_rmeta_1  253:4    0    4M  0 lvm
    │ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
    └─vg_var-lv_var_rimage_1 253:5    0  952M  0 lvm
      └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
    sde                        8:64   0    1G  0 disk
```
Удаляем временную VG для root
```
    lvremove /dev/vg_root/lv_root
    vgremove /dev/vg_root
    pvremove /dev/sdb
```
### **Выделить том для /home**
```
    lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
    mkfs.xfs /dev/VolGroup00/LogVol_Home
    mount /dev/VolGroup00/LogVol_Home /mnt/
    cp -aR /home/* /mnt/
    rm -rf /home/*
    umount /mnt
    mount /dev/VolGroup00/LogVol_Home /home/
    echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
Перезагружаемся, смотрим результат
```
    NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda                          8:0    0   40G  0 disk
    ├─sda1                       8:1    0    1M  0 part
    ├─sda2                       8:2    0    1G  0 part /boot
    └─sda3                       8:3    0   39G  0 part
      ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
      ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
      └─VolGroup00-LogVol_Home 253:7    0    2G  0 lvm  /home
    sdb                          8:16   0   10G  0 disk
    sdc                          8:32   0    2G  0 disk
    ├─vg_var-lv_var_rmeta_0    253:2    0    4M  0 lvm
    │ └─vg_var-lv_var          253:6    0  952M  0 lvm  /var
    └─vg_var-lv_var_rimage_0   253:3    0  952M  0 lvm
      └─vg_var-lv_var          253:6    0  952M  0 lvm  /var
    sdd                          8:48   0    1G  0 disk
    ├─vg_var-lv_var_rmeta_1    253:4    0    4M  0 lvm
    │ └─vg_var-lv_var          253:6    0  952M  0 lvm  /var
    └─vg_var-lv_var_rimage_1   253:5    0  952M  0 lvm
      └─vg_var-lv_var          253:6    0  952M  0 lvm  /var
    sde                          8:64   0    1G  0 disk
```
### **Сделать том для снэпшотов в /home**
Создадим файлики
```
     touch /home/file{1..20}
```
Сделаем снэпшот, удалим часть файлов
```
    lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
    rm -f /home/file{11..20}
```
Восстановимся из снэпшота
```
    umount /home
    vconvert --merge /dev/VolGroup00/home_snap
    mount /home
```
