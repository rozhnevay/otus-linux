# **Homework 2**

### **Добавим дисков**
Для этого пропишем дополнительные блоки в Vagrantfile
```
    :sata5 => {
        :dfile => './sata5.vdi',
        :size => 250, # Megabytes
    	:port => 5
    },
    :sata6 => {
            :dfile => './sata6.vdi',
            :size => 250,
            :port => 6
    },
    :sata7 => {
            :dfile => './sata7.vdi',
            :size => 250, # Megabytes
            :port => 7
    }
```
### **Создание RAID массивов**
Проверим, что диски подключены
```
    sudo lshw -short | grep disk
```
Результат
```
    [vagrant@otuslinux ~]$ sudo lshw -short | grep disk
    /0/100/1.1/0.0.0    /dev/sda   disk        42GB VBOX HARDDISK
    /0/100/d/0          /dev/sdb   disk        262MB VBOX HARDDISK
    /0/100/d/1          /dev/sdc   disk        262MB VBOX HARDDISK
    /0/100/d/2          /dev/sdd   disk        262MB VBOX HARDDISK
    /0/100/d/3          /dev/sde   disk        262MB VBOX HARDDISK
    /0/100/d/4          /dev/sdf   disk        262MB VBOX HARDDISK
    /0/100/d/5          /dev/sdg   disk        262MB VBOX HARDDISK
    /0/100/d/0.0.0      /dev/sdh   disk        262MB VBOX HARDDISK
```
Переключимся на суперпользователя для удобства
```
    sudo su
```
Зануляем суперблоки
```
    mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
```
Результат
```
    [root@otuslinux vagrant]# mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
    mdadm: Unrecognised md component device - /dev/sdb
    mdadm: Unrecognised md component device - /dev/sdc
    mdadm: Unrecognised md component device - /dev/sdd
    mdadm: Unrecognised md component device - /dev/sde
    mdadm: Unrecognised md component device - /dev/sdf

```
Создаем RAID-5 на 3-х дисках (необходимый минимум)
```
    mdadm --create --verbose /dev/md1 -l 5 -n 3 /dev/sd{d,e,f}
```
Результат
```
    [root@otuslinux vagrant]# mdadm --create --verbose /dev/md1 -l 5 -n 3 /dev/sd{d,e,f}
    mdadm: layout defaults to left-symmetric
    mdadm: layout defaults to left-symmetric
    mdadm: chunk size defaults to 512K
    mdadm: size set to 253952K
    mdadm: Defaulting to version 1.2 metadata
    mdadm: array /dev/md1 started.
```
Проверим, что RAID собрался:
```
    [root@otuslinux vagrant]# cat /proc/mdstat
    Personalities : [raid6] [raid5] [raid4]
    md1 : active raid5 sdf[3] sde[1] sdd[0]
          507904 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
    
    unused devices: <none>
```
```
    [root@otuslinux vagrant]# mdadm -D /dev/md1
    /dev/md1:
               Version : 1.2
         Creation Time : Tue May 10 05:01:22 2022
            Raid Level : raid5
            Array Size : 507904 (496.00 MiB 520.09 MB)
         Used Dev Size : 253952 (248.00 MiB 260.05 MB)
          Raid Devices : 3
         Total Devices : 3
           Persistence : Superblock is persistent
    
           Update Time : Tue May 10 05:01:27 2022
                 State : clean
        Active Devices : 3
       Working Devices : 3
        Failed Devices : 0
         Spare Devices : 0
    
                Layout : left-symmetric
            Chunk Size : 512K
    
    Consistency Policy : resync
    
                  Name : otuslinux:1  (local to host otuslinux)
                  UUID : 5e923913:f311f321:beb90227:3d06ca8c
                Events : 18
    
        Number   Major   Minor   RaidDevice State
           0       8       48        0      active sync   /dev/sdd
           1       8       64        1      active sync   /dev/sde
           3       8       80        2      active sync   /dev/sdf
```
### **Создание mdadm.conf**
Необходимо для того, чтобы ОС запомнила, какой RAID создавать при загрузке
Проверим информацию
```
    [root@otuslinux vagrant]# mdadm --detail --scan --verbose
    ARRAY /dev/md1 level=raid5 num-devices=3 metadata=1.2 name=otuslinux:1 UUID=5e923913:f311f321:beb90227:3d06ca8c
       devices=/dev/sdd,/dev/sde,/dev/sdf
```
Создадим mdadm.conf
```
    mkdir /etc/mdadm
    echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
    sudo mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf   
```
### **Ломаем/чиним RAID**
```
    [root@otuslinux vagrant]# mdadm /dev/md1 --fail /dev/sdd
    mdadm: set /dev/sdd faulty in /dev/md1
```
Проверяем, что точно сломали
```
    [root@otuslinux vagrant]# cat /proc/mdstat
    Personalities : [raid6] [raid5] [raid4]
    md1 : active raid5 sdf[3] sdd[0](F) sde[1]
          507904 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [_UU]
    
    unused devices: <none>
```
Удаляем сломанный диск из массива
```
    [root@otuslinux vagrant]# mdadm /dev/md1 --remove /dev/sdd
    mdadm: hot removed /dev/sdd from /dev/md1
```
Добавляем новый диск и смотрим на процесс rebuilding:
```
    [root@otuslinux vagrant]# mdadm /dev/md1 --add /dev/sdd
    mdadm: added /dev/sdd
    [root@otuslinux vagrant]# cat /proc/mdstat
    Personalities : [raid6] [raid5] [raid4]
    md1 : active raid5 sdd[4] sdf[3] sde[1]
          507904 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [_UU]
          [===========>.........]  recovery = 59.1% (150528/253952) finish=0.0min speed=75264K/sec
    
    unused devices: <none>
```
После окончания процесса:
```
    [root@otuslinux vagrant]# cat /proc/mdstat
    Personalities : [raid6] [raid5] [raid4]
    md1 : active raid5 sdd[4] sdf[3] sde[1]
          507904 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
    
    unused devices: <none>
```
### **Создаем GPT раздел**
Маркируем наш RAID как GPT
```
    parted -s /dev/md1 mklabel gpt
```
Создаем партиции
```
    parted /dev/md1 mkpart primary ext4 0% 20%
    parted /dev/md1 mkpart primary ext4 20% 40%
    parted /dev/md1 mkpart primary ext4 40% 60%
    parted /dev/md1 mkpart primary ext4 60% 80%
    parted /dev/md1 mkpart primary ext4 80% 100%
```
Создаем файловую систему на этих партициях
```
    for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md1p$i; done
```
Монтируем партиции по каталогам
```
    for i in $(seq 1 5); do mount /dev/md1p$i /raid/part$i; done
```
