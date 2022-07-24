# **Homework 15**
Добавим двух пользователей admin и weekdays и установим пароль для них.
Убедимся, что включена аутентификация по паролю
```shell
[root@80-78-247-43 ~]# useradd admin
[root@80-78-247-43 ~]# id admin
uid=1000(admin) gid=1000(admin) groups=1000(admin)
[root@80-78-247-43 ~]# useradd weekdays
[root@80-78-247-43 ~]# echo XXXXX | sudo passwd --stdin admin
Changing password for user admin.
passwd: all authentication tokens updated successfully.
[root@80-78-247-43 ~]# echo XXXXX | sudo passwd --stdin weekdays
Changing password for user weekdays.
passwd: all authentication tokens updated successfully.
[root@80-78-247-43 ~]# vi /etc/ssh/sshd_config 
```
Отредактируем /etc/security/time.conf, добавим туда
```shell
*;*;weekdays;Wd
```
Добавим в /etc/pam.d/sshd:
```shell
account    required     pam_time.so
```
Сегодня воскресенье. Попробуем подключиться под weekdays:
```shell
*;*;!admin;!Wd
```
Подключимся под weekdays и видим, что нас не пускает:
```shell
ssh weekdays@89.108.70.33
weekdays@89.108.70.33's password: 
Connection closed by 89.108.70.33 port 22
```
Подключимся под admin и видим, что пускает:
```shell
ssh admin@89.108.70.33
admin@89.108.70.33's password: 
Last failed login: Sun Jul 24 18:17:57 MSK 2022 from 119.167.219.132 on ssh:notty
There were 9886 failed login attempts since the last successful login.
[admin@89-108-70-33 ~]$
```



