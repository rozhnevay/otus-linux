# **Homework 4**
Переключимся на суперпользователя
```
    sudo su
```

### **Определение алгоритма с наилучшим сжатием**
Смотрим какие у нас есть диски
```
    lsblk
```
```
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda      8:0    0   40G  0 disk
    └─sda1   8:1    0   40G  0 part /
    sdb      8:16   0  512M  0 disk
    sdc      8:32   0  512M  0 disk
    sdd      8:48   0  512M  0 disk
    sde      8:64   0  512M  0 disk
    sdf      8:80   0  512M  0 disk
    sdg      8:96   0  512M  0 disk
    sdh      8:112  0  512M  0 disk
    sdi      8:128  0  512M  0 disk
```
Создаем пулы из двух дисков в режиме RAID1
```
    zpool create otus1 mirror /dev/sdb /dev/sdc
    zpool create otus2 mirror /dev/sdd /dev/sde
    zpool create otus3 mirror /dev/sdf /dev/sdg
    zpool create otus4 mirror /dev/sdh /dev/sdi
```
```
    zpool list
```
```
    NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
    otus1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
    otus2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
    otus3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
    otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
```
Добавим разные алгоритмы сжатия на каждую файловую систему
```
    zfs set compression=lzjb otus1
    zfs set compression=lz4 otus2
    zfs set compression=gzip-9 otus3
    zfs set compression=zle otus4
```
```
    zfs get all | grep compression
```
Убедимся, что разные ФС имеют разные алгоритмы сжатия
```
    otus1  compression           lzjb                   local
    otus2  compression           lz4                    local
    otus3  compression           gzip-9                 local
    otus4  compression           zle                    local
```
Скачаем "Война и мир" и проверим степень сжатия
```
    for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
    zfs list
    zfs get all | grep compressratio | grep -v ref
```
Лучше всего gzip-9
```
    otus1  compressratio         1.81x                  -
    otus2  compressratio         2.22x                  -
    otus3  compressratio         3.64x                  -
    otus4  compressratio         1.00x                  -
```

### **Определение настроек пула**
Загрузим архив и разархивируем его
```
    wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download'
    tar -xzvf archive.tar.gz
```
Проверм возможность импорта в пул
```
    zpool import -d zpoolexport/
```
```
       pool: otus
         id: 6554193320433390805
      state: ONLINE
     action: The pool can be imported using its name or numeric identifier.
     config:
    
            otus                         ONLINE
              mirror-0                   ONLINE
                /root/zpoolexport/filea  ONLINE
                /root/zpoolexport/fileb  ONLINE

```
Импортируем
```
    zpool import -d zpoolexport/ otus
    zpool status
```
```
    pool: otus
     state: ONLINE
      scan: none requested
    config:
    
            NAME                         STATE     READ WRITE CKSUM
            otus                         ONLINE       0     0     0
              mirror-0                   ONLINE       0     0     0
                /root/zpoolexport/filea  ONLINE       0     0     0
                /root/zpoolexport/fileb  ONLINE       0     0     0
    
    errors: No known data errors

```
Определяем настройки
```
    zfs get available otus -H >> settings
    zfs get type otus -H >> settings
    zfs get recordsize otus -H >> settings
    zfs get compression otus -H >> settings
    zfs get checksum otus -H >> settings
```
```
    otus    available       350M    -
    otus    available       350M    -
    otus    type    filesystem      -
    otus    recordsize      128K    local
    otus    compression     zle     local
    otus    checksum        sha256  local
```

### **Работа со снапшотом, поиск сообщения от преподавателя**
```
    wget -O otus_task2.file --no-check-certificate 'https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download'
    zfs receive otus/test@today < otus_task2.file
    cat /otus/test/task1/file_mess/secret_message
```
Секретное сообщение
```
    https://github.com/sindresorhus/awesome
```
