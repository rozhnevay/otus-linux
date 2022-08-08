# **Homework 17**
Настраиваемые бэкапы

Подключим диск 2Гб к /var/backup
Посмотрим, какие есть диски:
```shell script
[root@zfs ~]# lsblk
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda      8:0    0  40G  0 disk
└─sda1   8:1    0  40G  0 part /
sdb      8:16   0   2G  0 disk
```
Подготовим lv на наш диск 2Гб
```shell script
    yum install -y lvm2
    pvcreate /dev/sdb
    vgcreate vg_root /dev/sdb
    lvcreate -n lv_root -l +100%FREE /dev/vg_root
    mkfs.xfs /dev/vg_root/lv_root
```
Подмонтируем lv_root в /var/backup
```shell script
     mkdir /var/backup
     mount /dev/vg_root/lv_root /var/backup
```
На client и backup устанавливаем borgbackup
```shell script
yum install epel-release
yum install borgbackup
```
На backup создаем пользователя и назначаем его на /var/backup
```shell script
useradd -m borg
chown borg:borg /var/backup/
```
Создаем каталог authorized_keys
```shell script
su - borg
mkdir .ssh
touch .ssh/authorized_keys
chmod 700 .ssh
chmod 600 .ssh/authorized_keys
```
Генерим ключ на client и добавляем в authorized_keys
```shell script
ssh-keygen
```
Инициализируем репозиторий borg с client:
```shell script
borg init --encryption=repokey borg@192.168.50.15:/var/backup/
```
Запускаем для проверки создания бэкапа и смотрим результат
```shell script
borg create --stats --list borg@192.168.50.15:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc
borg list borg@192.168.50.15:/var/backup/
```
```shell script
etc-2022-08-08_06:05:31              Mon, 2022-08-08 06:05:35 [b816ba82e8bc4b4183696d77a23741a2ea94f7e62e44b48d5a936da0747051a7]
```
Смотрим список файлов
```shell script
borg list borg@192.168.50.15:/var/backup/::etc-2022-08-08_06:05:31
```
Автоматизируем создание бэкапов
```shell script
vi /etc/systemd/system/borg-backup.service
```
```shell script
[Unit]
Description=Borg Backup
[Service]
Type=oneshot
# Парольная фраза
Environment="BORG_PASSPHRASE=****"
# Репозиторий
Environment=REPO=borg@192.168.50.15:/var/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc
# Создание бэкапа
ExecStart=/bin/borg create \
--stats \
${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}
# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}
# Очистка старых бэкапов
ExecStart=/bin/borg prune \
--keep-daily 90 \
--keep-monthly 12 \
--keep-yearly 1 \
${REPO}
```
vi /etc/systemd/system/borg-backup.timer
```shell script
# /etc/systemd/system/borg-backup.timer
[Unit]
Description=Borg Backup
[Timer]
OnUnitActiveSec=5min
[Install]
WantedBy=timers.target
```
```shell script
systemctl enable borg-backup.timer
systemctl start borg-backup.timer
```
Статус таймера и бэкапы:
```shell script
[root@client ~]# systemctl list-timers --all
NEXT                         LEFT      LAST                         PASSED   UNIT                         ACTIVATES
Mon 2022-08-08 06:17:07 UTC  934ms ago Mon 2022-08-08 06:17:08 UTC  3ms ago  borg-backup.timer            borg-backup.service
```
```shell script
[root@client ~]# borg list borg@192.168.50.15:/var/backup/
Enter passphrase for key ssh://borg@192.168.50.15/var/backup:
etc-2022-08-08_06:25:11              Mon, 2022-08-08 06:25:11 [ca107a35bbd213ad623c4977fb205fef70b8f66dbf347f7ef67d41de4ce4825c]
```
Удалим /etc/hostname и восстановим из бэкапа:
```shell script
rm -f /etc/hostname

[root@client ~]# borg extract borg@192.168.50.15:/var/backup/::etc-2022-08-08_06:25:11 ^Cc/hostname
[root@client ~]# ll
total 16
-rw-------. 1 root root 5570 Apr 30  2020 anaconda-ks.cfg
drwx------. 2 root root   22 Aug  8 06:28 etc
-rw-------. 1 root root 5300 Apr 30  2020 original-ks.cfg
[root@client ~]# cd etc/
[root@client etc]# ll
total 4
-rw-r--r--. 1 root root 7 Aug  8 05:53 hostname
[root@client etc]# mv hostname /etc/hostname
[root@client etc]#
```
Успешно восстановили
