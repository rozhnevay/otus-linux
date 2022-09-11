# **Homework 23**
### DNS

Запустим vagrant up со скриптами Ansible.
Скорректируем /etc/resolv.conf для DNS-серверов
На ns01 указываем nameserver 192.168.50.10.
На ns02 указываем nameserver 192.168.50.11.

На DNS серверах допишем в /etc/named/named.dns.lab допишем:
```text
;Web
web1    IN  A   192.168.50.15
web2    IN  A   192.168.50.16
```

```shell
systemctl restart named
```
Сделам +1 к serial.
Выполним проверку с клиента:
```shell
[root@client ~]# dig @192.168.50.10 web1.dns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> @192.168.50.10 web1.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60837
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;web1.dns.lab.                  IN      A

;; ANSWER SECTION:
web1.dns.lab.           3600    IN      A       192.168.50.15
```
### Создание новой зоны и добавление в неё записей
Добавим в файл /etc/named.conf
На ns01
```text
// lab's newdns zone
zone "newdns.lab" {
type master;
allow-transfer { key "zonetransfer.key"; };
allow-update { key "zonetransfer.key"; };
file "/etc/named/named.newdns.lab";
};
```
На ns02
```text
// lab's newdns zone
zone "newdns.lab" {
type slave;
masters { 192.168.50.10; };
file "/etc/named/named.newdns.lab";
};
```
На ns01 создадим /etc/named/named.newdns.lab
```text
$TTL 3600
$ORIGIN newdns.lab.
@   IN  SOA ns01.dns.lab. root.dns.lab. (
            2711201007  ; serial
            3600        ; refresh (1 hour)
            600         ; retry (10 minutes)
            86400       ; expire (1 day)
            600         ; minimum (10 minutes)
        )
    IN  NS  ns01.dns.lab.
    IN  NS  ns02.dns.lab.
; DNS Servers
ns01    IN  A   192.168.50.10
ns02    IN  A   192.168.50.11

;WWW
www IN  A   192.168.50.15
www IN  A   192.168.50.16
```
## Настройка Split-DNS
Создадим /etc/named/named.dns.lab.client
```text
$TTL 3600
$ORIGIN dns.lab.
@   IN  SOA ns01.dns.lab. root.dns.lab. (
            2711201407  ; serial
            3600        ; refresh (1 hour)
            600         ; retry (10 minutes)
            86400       ; expire (1 day)
            600         ; minimum (10 minutes)
        )
    IN  NS  ns01.dns.lab.
    IN  NS  ns02.dns.lab.
; DNS Servers
ns01    IN  A   192.168.50.10
ns02    IN  A   192.168.50.11

;WWW
web IN  A   192.168.50.15
```
Сгенерируем ключи для client и client2
```shell
tsig-keygen
```
Добавим их в конец /etc/named.conf
```text
#Описание ключа для хоста client
key "client-key" {
algorithm hmac-sha256;
secret "IQg171Ht4mdGYcjjYKhI9gSc1fhoxzHZB+h2NMtyZWY=";
};
#Описание ключа для хоста client2
key "client2-key" {
algorithm hmac-sha256;
secret "m7r7SpZ9KBcA4kOl1JHQQnUiIlpQA1IJ9xkBHwdRAHc=";
};
#Описание access-листов
acl client { !key client2-key; key client-key; 192.168.50.15; };
acl client2 { !key client-key; key client2-key; 192.168.50.16; };
```
Вносим изменения на ns02 подобно ns01 только в настройках будет указание забирать
информацию с сервера ns01

Проверка на client:
```shell
[root@client ~]# ping www.newdns.lab
PING www.newdns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.014 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.066 ms
^C
--- www.newdns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.014/0.040/0.066/0.026 ms
[root@client ~]# ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.015 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.068 ms
^C
--- web1.dns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1005ms
rtt min/avg/max/mdev = 0.015/0.041/0.068/0.027 ms
[root@client ~]# ping web2.dns.lab
ping: web2.dns.lab: Name or service not known

Проверка на client2:
[root@client2 ~]# ping www.newdns.lab
ping: www.newdns.lab: Name or service not known
[root@client2 ~]#
[root@client2 ~]# ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=1 ttl=64 time=0.809 ms
^C
--- web1.dns.lab ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.809/0.809/0.809/0.000 ms
[root@client2 ~]# ping web2.dns.lab
PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.
64 bytes from client2 (192.168.50.16): icmp_seq=1 ttl=64 time=0.037 ms
64 bytes from client2 (192.168.50.16): icmp_seq=2 ttl=64 time=0.065 ms
```