# **Homework 16**
Централизованный сбор логов

Установим часовой пояс мск на всех серверах
```shell script
cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime
systemctl restart chronyd
systemctl status chronyd
```
Установим nginx
```shell script
yum install epel-release
yum install -y nginx
systemctl status nginx
```
Настроим сервер логов
Проверим наличие rsyslog
```shell script
[root@log ~]# yum list rsyslog
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: ftp.antilo.de
 * extras: mirror.softaculous.com
 * updates: ftp.rz.uni-frankfurt.de
base                                                                                                                                                                                        | 3.6 kB  00:00:00
extras                                                                                                                                                                                      | 2.9 kB  00:00:00
updates                                                                                                                                                                                     | 2.9 kB  00:00:00
(1/4): base/7/x86_64/group_gz                                                                                                                                                               | 153 kB  00:00:00
(2/4): extras/7/x86_64/primary_db                                                                                                                                                           | 247 kB  00:00:01
(3/4): base/7/x86_64/primary_db                                                                                                                                                             | 6.1 MB  00:00:01
(4/4): updates/7/x86_64/primary_db                                                                                                                                                          |  16 MB  00:00:05
Installed Packages
rsyslog.x86_64                                                                                     8.24.0-52.el7                                                                                          @anaconda
Available Packages
rsyslog.x86_64                                                                                     8.24.0-57.el7_9
```
Внесем изменения в /etc/rsyslog.conf
```shell script
# Provides UDP syslog reception
$ModLoad imudp
$UDPServerRun 514

# Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514

*****
#Add remote logs
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
```
Перезапустим службу и проверим, что порты прослушиваются
```shell script
systemctl restart rsyslog
ss -tuln | grep 514
```
```shell script
[root@log ~]# ss -tuln | grep 514
udp    UNCONN     0      0         *:514                   *:*
udp    UNCONN     0      0      [::]:514                [::]:*
tcp    LISTEN     0      25        *:514                   *:*
tcp    LISTEN     0      25     [::]:514                [::]:*
```
Настраиваем отправку логов на web-сервере. Редактируем nginx.conf
```shell script
error_log /var/log/nginx/error.log;
error_log syslog:server=192.168.50.15:514,tag=nginx_error debug;
access_log syslog:server=192.168.50.15:514,tag=nginx_access,severity=info combined;
```
Перезапустим nginx и пробуем несколько раз дернуть url. Удалим index.html, чтобы зафиксировались error-логи.
Смотрим логи на log-сервере
```shell script
[root@log ~]# cat /var/log/rsyslog/web/nginx_access.log
Aug  7 18:25:30 web nginx_access: ::1 - - [07/Aug/2022:18:25:30 +0300] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0"
Aug  7 18:26:01 web nginx_access: ::1 - - [07/Aug/2022:18:26:01 +0300] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0"
Aug  7 18:26:12 web nginx_access: ::1 - - [07/Aug/2022:18:26:12 +0300] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0"
[root@log web]# cat /var/log/rsyslog/web/nginx_error.log
Aug  7 18:31:54 web nginx_error: 2022/08/07 18:31:54 [error] 3462#3462: *5 directory index of "/usr/share/nginx/html/" is forbidden, client: ::1, server: _, request: "GET / HTTP/1.1", host: "localhost"
Aug  7 18:32:30 web nginx_error: 2022/08/07 18:32:30 [error] 3462#3462: *6 directory index of "/usr/share/nginx/html/" is forbidden, client: ::1, server: _, request: "GET / HTTP/1.1", host: "localhost"
Aug  7 18:32:30 web nginx_error: 2022/08/07 18:32:30 [error] 3462#3462: *7 directory index of "/usr/share/nginx/html/" is forbidden, client: ::1, server: _, request: "GET / HTTP/1.1", host: "localhost"
Aug  7 18:36:12 web nginx_error: 2022/08/07 18:36:12 [error] 3627#3627: *1 directory index of "/usr/share/nginx/html/" is forbidden, client: ::1, server: _, request: "GET / HTTP/1.1", host: "localhost"
Aug  7 18:39:09 web nginx_error: 2022/08/07 18:39:09 [debug] 3647#3647: epoll add event: fd:5 op:1 ev:00002001
Aug  7 18:39:09 web nginx_error: 2022/08/07 18:39:09 [debug] 3647#3647: epoll add event: fd:6 op:1 ev:00002001
Aug  7 18:39:13 web nginx_error: 2022/08/07 18:39:13 [debug] 3647#3647: accept on [::]:80, ready: 0
Aug  7 18:39:13 web nginx_error: 2022/08/07 18:39:13 [debug] 3647#3647: posix_memalign: 000056299F60AC00:512 @16
Aug  7 18:39:13 web nginx_error: 2022/08/07 18:39:13 [debug] 3647#3647: *1 accept: [::1]:34742 fd:3
Aug  7 18:39:13 web nginx_error: 2022/08/07 18:39:13 [debug] 3647#3647: *1 event timer add: 3: 60000:2522761
Aug  7 18:39:13 web nginx_error: 2022/08/07 18:39:13 [debug] 3647#3647: *1 reusable connection: 1
Aug  7 18:39:13 web nginx_error: 2022/08/07 18:39:13 [debug] 3647#3647: *1 epoll add event: fd:3 op:1 ev:80002001
Aug  7 18:39:13 web nginx_error: 2022/08/07 18:39:13 [debug] 3647#3647: *1 http wait request handler
```
Настройка аудита, контролирующего изменения конфигурации nginx
Проверяем наличие утилиты audit
```shell script
[root@web nginx]# rpm -qa | grep audit
audit-2.8.5-4.el7.x86_64
audit-libs-2.8.5-4.el7.x86_64
```
Отредактируем /etc/audit/rules.d/audit.rules, добавим в конце:
```shell script
-w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d/ -p wa -k nginx_conf
```
Перезапустим службу auditd
```shell script
service auditd restart
```
Внесем изменения в nginx.conf - закомментируем оотправку error_log на удаленный сервер и проверим, записалось ли это событие в audit:
```shell script
ausearch -f /etc/nginx/nginx.conf
```
```shell script
----
time->Sun Aug  7 20:08:41 2022
type=CONFIG_CHANGE msg=audit(1659892121.038:1141): auid=1000 ses=8 op=updated_rules path="/etc/nginx/nginx.conf" key="nginx_conf" list=4 res=1
----
time->Sun Aug  7 20:08:41 2022
type=PROCTITLE msg=audit(1659892121.038:1142): proctitle=7669002F6574632F6E67696E782F6E67696E782E636F6E66
type=PATH msg=audit(1659892121.038:1142): item=1 name="/etc/nginx/nginx.conf" inode=12491 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=unconfined_u:object_r:httpd_config_t:s0 objtype=CRE
ATE cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
type=PATH msg=audit(1659892121.038:1142): item=0 name="/etc/nginx/" inode=85 dev=08:01 mode=040755 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=PARENT cap_fp=0000000
000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
type=CWD msg=audit(1659892121.038:1142):  cwd="/var/log/nginx"
type=SYSCALL msg=audit(1659892121.038:1142): arch=c000003e syscall=2 success=yes exit=3 a0=1d50750 a1=241 a2=1a4 a3=0 items=2 ppid=3501 pid=22445 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egi
d=0 sgid=0 fsgid=0 tty=pts0 ses=8 comm="vi" exe="/usr/bin/vi" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
----
```
