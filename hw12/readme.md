# **Homework 12**
Запускаем стенд, видим ошибку запуска nginx - permission denied от SELinux
```
        selinux: Jul 03 15:14:16 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
        selinux: Jul 03 15:14:16 selinux nginx[2989]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
        selinux: Jul 03 15:14:16 selinux nginx[2989]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
```
## **Запуск nginx на нестандартном порту тремя способами**
### Способ 1 - сброс булевского параметра nis_enabled
Проверим, что отключен файервол
```shell script
systemctl status firewalld
```
```
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
```
Проверим конфигурацию nginx
```shell script
[root@selinux vagrant]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Проверим режим работы SELinux
```shell script
[root@selinux vagrant]# getenforce
Enforcing
```
Найдем в аудите информацию о блокировании порта
```
[root@selinux vagrant]# cat /var/log/audit/audit.log | grep 4881
type=AVC msg=audit(1656861256.844:850): avc:  denied  { name_bind } for  pid=2989 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```
Парсим с помощью audit2why
```
[root@selinux vagrant]# cat /var/log/audit/audit.log | grep 4881 | audit2why
type=AVC msg=audit(1656861256.844:850): avc:  denied  { name_bind } for  pid=2989 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```

Видим подсказку, что нужно включить nis_enabled, включаем и перегружаем nginx
```shell script
  setsebool -P nis_enabled 1
  systemctl restart nginx
  systemctl status nginx
```
Все ок
```
[root@selinux vagrant]#   systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-07-03 15:25:40 UTC; 1s ago
  Process: 3253 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
```
Чекнем курлом:
```shell script
curl http://localhost:4881
```
```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css">

        html {
        background-image:url(img/html-background.png);
        background-color: white;
        font-family: "DejaVu Sans", "Liberation Sans", sans-serif;
        font-size: 0.85em;
        line-height: 1.25em;
        margin: 0 4% 0 4%;
        }
```
Проверим статус параметра
```shell script
getsebool -a | grep nis_enabled
```
```
[root@selinux vagrant]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
Сбросим параметр
```shell script
setsebool -P nis_enabled 0
```
### Способ 2 - добавление порта в тип
Ищем тип для http-трафика
```shell script
semanage port -l | grep http
```
```
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавим порт в тип http_port_t
```shell script
semanage port -a -t http_port_t -p tcp 4881
```
Проверим теперь
```shell script
[root@selinux vagrant]# semanage port -l | grep http
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
```
Перезапустим nginx и проверим, поднялся ли. Видим, что все ок
```shell script
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-07-03 15:49:02 UTC; 6s ago
```
Удалим нестандартный порт и убедимся, что nginx не запустится
```shell script
semanage port -d -t http_port_t -p tcp 4881
```
```shell script
[root@selinux vagrant]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sun 2022-07-03 15:50:49 UTC; 3s ago
  Process: 3300 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3321 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 3320 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3302 (code=exited, status=0/SUCCESS)

Jul 03 15:50:49 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jul 03 15:50:49 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 03 15:50:49 selinux nginx[3321]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jul 03 15:50:49 selinux nginx[3321]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
Jul 03 15:50:49 selinux nginx[3321]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jul 03 15:50:49 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
Jul 03 15:50:49 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
```
### Способ 3 - установка модуля SELinux
С помощью утилиты audit2allow посмотрим модуль, разрешающий nginx работу на нестандартном порту
```shell script
grep nginx /var/log/audit/audit.log | audit2allow -M nginx
```
```shell script
[root@selinux vagrant]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
Применим модуль
```shell script
semodule -i nginx.pp
```
Перегружаем nginx, смотрим статус
```shell script
[root@selinux vagrant]# semodule -i nginx.pp
[root@selinux vagrant]# systemctl restart nginx
[root@selinux vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-07-03 15:54:22 UTC; 3s ago
  Process: 3349 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
```
Просмотр всех модулей
```shell script
semodule -l
```

Удаление модуля
```shell script
semodule -r nginx
```
## **Обеспечение работоспособности приложения при включенном SELinux**
Клонируем репу и запускаем машины.
Подключаемся к client и пытаемся добавить dns зону.
```shell script
nsupdate -k /etc/named.zonrtransfer.key
```
```shell script
[root@client vagrant]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
```
Проверим логи SELinux на клиенте
```shell script
[root@client vagrant]# cat /var/log/audit/audit.log | audit2why
[root@client vagrant]#
```
На клиенте нет ошибок, подключаемся к серверу и проверяем логи
```shell script
[root@ns01 vagrant]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1656910367.573:1927): avc:  denied  { create } for  pid=5213 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```
Видим, что вместо типа *named_t* используется тип *etc_t*.
Проверим каталог /etc/named
```shell script
ls -laZ /etc/named
```
```shell script
[root@ns01 vagrant]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```
Контекст неправильный - конфиг файлы лежат в другом каталоге.
Чтобы понять, где они должны лежать, используем команду:
```shell script
semanage fcontext -l | grep named
```
```shell script
[root@ns01 vagrant]# semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0
```
Изменим тип контекста безопасности для /etc/named:
```shell script
chcon -R -t named_zone_t /etc/named
```
```shell script
ls -laZ /etc/named
```
```shell script
[root@ns01 vagrant]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```
Попробуем внести изменения с клиента снова
Теперь все ок
```shell script
[root@client vagrant]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```
Снова тестируем и получаем успешный ответ
```shell script
[root@client vagrant]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51541
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 16 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Mon Jul 04 05:21:15 UTC 2022
;; MSG SIZE  rcvd: 96
```
Сделаем reboot хостов и попробуем еще раз. Видим, что все настройки сохранились, команда проходит:
```shell script
[root@client vagrant]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62235
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 2 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Mon Jul 04 05:23:26 UTC 2022
;; MSG SIZE  rcvd: 96
```
Возвращаем обратно настройки на DNS-сервере
```
restorecon -v -R /etc/named
```
```shell script
[root@ns01 vagrant]# restorecon -v -R /etc/named
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
```
