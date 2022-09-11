# **Homework 22**
### VPN

Установим openvpn и отключим selinux на обоих серверах
```shell
yum install -y epel-release
yum install -y openvpn iperf3
setenforce 0
```
На серверем произведем настройку
Создадим файл-ключ
```shell
openvpn --genkey --secret /etc/openvpn/static.key
```
## Настройка TAP-сервера
Делаем конфиг файл vi /etc/openvpn/server.conf
Его содержимое для TAP-режима
```text
dev tap
ifconfig 10.10.10.1 255.255.255.0
topology subnet
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
Запускаем openvpn сервер и добавляем в автозагрузку
```shell
systemctl start openvpn@server
systemctl enable openvpn@server
```
## Настройка клиента
Делаем конфиг файл vi /etc/openvpn/server.conf
```shell
dev tap
remote 192.168.10.10
ifconfig 10.10.10.2 255.255.255.0
topology subnet
route 192.168.10.0 255.255.255.0
secret /etc/openvpn/static.key
comp-lzo
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
Скопируем файл ключ static.key в /etc/openvpn/
```shell
systemctl start openvpn@server
systemctl enable openvpn@server
```
Замеряем скорость в туннеле.
На сервере:
```shell
iperf3 -s &
```
На клиенте
```shell
iperf3 -c 10.10.10.1 -t 40 -i 5
```
Получили результат
```text
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-5.00   sec   127 MBytes   214 Mbits/sec   40    777 KBytes       
[  4]   5.00-10.00  sec   132 MBytes   222 Mbits/sec    0    854 KBytes       
[  4]  10.00-15.00  sec   132 MBytes   222 Mbits/sec    4    386 KBytes       
^C[  4]  15.00-17.30  sec  59.9 MBytes   219 Mbits/sec    4    182 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-17.30  sec   452 MBytes   219 Mbits/sec   48             sender
[  4]   0.00-17.30  sec  0.00 Bytes  0.00 bits/sec                  receiver
iperf3: interrupt - the client has terminated
```
219 Mbit/s - пропускная способность в режиме tap
Повторяем шаги для tun, меняем dev-директиву, и замеряем:
```text
Connecting to host 10.10.10.1, port 5201
[  4] local 10.10.10.2 port 47550 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-5.01   sec   132 MBytes   222 Mbits/sec   19    497 KBytes       
[  4]   5.01-10.01  sec   134 MBytes   225 Mbits/sec    1    489 KBytes       
[  4]  10.01-15.00  sec   131 MBytes   220 Mbits/sec    1    461 KBytes       
^C[  4]  15.00-19.07  sec   100 MBytes   206 Mbits/sec    0    597 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-19.07  sec   498 MBytes   219 Mbits/sec   21             sender
[  4]   0.00-19.07  sec  0.00 Bytes  0.00 bits/sec                  receiver
iperf3: interrupt - the client has terminated
```
Вывод - скорость примерно одинаковая, но retry в два раза больше в режиме TAP.

## RAS на базе OpenVPN
```shell
yum install -y epel-release
yum install -y openvpn easy-rsa
setenforce 0
```
Инициализируем PKI
```shell
cd /etc/openvpn/
/usr/share/easy-rsa/3.0.8/easyrsa init-pki
```
Сгенерируем ключи и серты для сервера
```shell
echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa build-ca nopass
echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa gen-req server nopass
echo 'yes' | /usr/share/easy-rsa/3.0.8/easyrsa sign-req server server
/usr/share/easy-rsa/3.0.8/easyrsa gen-dh
openvpn --genkey --secret ta.key
```
Сгенерируем серты для клиента
```shell
echo 'client' | /usr/share/easy-rsa/3/easyrsa gen-req client nopass
echo 'yes' | /usr/share/easy-rsa/3/easyrsa sign-req client client
```
Создадим конфиг файл /etc/openvpn/server.conf
```text
port 1207
proto udp
dev tun
ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/issued/server.crt
key /etc/openvpn/pki/private/server.key
dh /etc/openvpn/pki/dh.pem
server 10.10.10.0 255.255.255.0
route 192.168.10.0 255.255.255.0
push "route 192.168.10.0 255.255.255.0"
ifconfig-pool-persist ipp.txt
client-to-client
client-config-dir /etc/openvpn/client
keepalive 10 120
comp-lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```
```shell
echo 'iroute 192.168.33.0 255.255.255.0' > /etc/openvpn/client/client
systemctl start openvpn@server
systemctl enable openvpn@server
```
Скопируем серты/ключи на хост машину
/etc/openvpn/pki/ca.crt
/etc/openvpn/pki/issued/client.crt
/etc/openvpn/pki/private/client.key
```shell
vagrant ssh -c "sudo cat /etc/openvpn/pki/ca.crt" > config/ca.crt
vagrant ssh -c "sudo cat /etc/openvpn/pki/issued/client.crt" > config/client.crt
vagrant ssh -c "sudo cat /etc/openvpn/pki/private/client.key" > config/client.key
```
Подключаем к openvpn с хост-машины
```shell
openvpn --config client.conf
```
```shell
2022-09-11 17:56:33 --cipher is not set. Previous OpenVPN version defaulted to BF-CBC as fallback when cipher negotiation failed in this case. If you need this fallback please add '--data-ciphers-fallback BF-CBC' to your configuration and/or add BF-CBC to --data-ciphers.
2022-09-11 17:56:33 WARNING: file './client.key' is group or others accessible
2022-09-11 17:56:33 OpenVPN 2.5.5 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Mar 22 2022
2022-09-11 17:56:33 library versions: OpenSSL 3.0.2 15 Mar 2022, LZO 2.10
2022-09-11 17:56:33 WARNING: No server certificate verification method has been enabled.  See http://openvpn.net/howto.html#mitm for more info.
2022-09-11 17:56:33 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.10.10:1207
2022-09-11 17:56:33 Socket Buffers: R=[212992->212992] S=[212992->212992]
2022-09-11 17:56:33 UDP link local (bound): [AF_INET][undef]:1194
2022-09-11 17:56:33 UDP link remote: [AF_INET]192.168.10.10:1207
```

Проверяем, пинг проходит до 10.10.10.1