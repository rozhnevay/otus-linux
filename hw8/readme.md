# **Homework 8**
Переключимся на суперпользователя
```
    sudo su
```

### **Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig)**
Создаем файл с настройками
```bash
    vi /etc/sysconfig/log-otus
```
```bash
    LOG_FILE="/var/log/log-otus"
    LOG_SEARCH="OTUS"
```
Создаем unit-файл и соответствующий timer-файл
```bash
    vi /etc/systemd/system/log-otus.service
```
```
    [Unit]
    Description=Otus log service
    
    [Service]
    Type=oneshot
    EnvironmentFile=/etc/sysconfig/log-otus
    ExecStart=/usr/local/bin/otus-log.sh "$LOG_FILE" "$LOG_SEARCH"
    
    [Install]
    WantedBy=multi-user.target
```
```bash
    vi /etc/systemd/system/log-otus.timer
```
```
    [Unit]
    Description=Otus log timer
    
    [Timer]
    OnCalendar=*:*:0/30
    AccuracySec=1s
    
    [Install]
    WantedBy=multi-user.target
```
Создадим скрипт для чтения файла /usr/local/bin/otus-log.sh
```bash
    vi /usr/local/bin/otus-log.sh
```
```bash
    #!/usr/bin/env bash
    
    LOG_FILE="$1"
    LOG_SEARCH="$2"
    LOG_POS_FILE="/tmp/log-otus.pos"
    
    [[ ! -f "$LOG_FILE" ]] && exit 1
    
    WC_FILE=$(wc -l "$LOG_FILE")
    WC_FILE=${WC_FILE%% *}
    
    [[ -f "$LOG_POS_FILE" ]] && LAST_POS=$(cat "$LOG_POS_FILE") || LAST_POS=0
    POS=$((WC_FILE - LAST_POS))
    [[ $POS -lt 0 ]] && POS=0
    
    (tail -"$POS" "$LOG_FILE" | grep -q "$LOG_SEARCH")
    RET=$?
    
    echo -en "$WC_FILE" >"$LOG_POS_FILE"
    
    [[ $RET -eq 0 ]] && echo "Found \"$LOG_SEARCH\" in file $LOG_FILE" | systemd-cat -t log-otus
    
    exit 0
```
Устанавливаем пермиссии на исполняемый файл
```
    chmod o+x /usr/local/bin/otus-log.sh
```
Делаем daemon-reload и запускаем наш unit
```
    systemctl daemon-reload
    systemctl enable log-otus.timer
    systemctl status log-otus.timer
```
```
    [root@kernel-update vagrant]# systemctl status log-otus.timer
    ● log-otus.timer - Otus log timer
       Loaded: loaded (/etc/systemd/system/log-otus.timer; enabled; vendor preset: disabled)
       Active: active (waiting) since Mon 2022-06-06 05:24:48 UTC; 1s ago
    
    Jun 06 05:24:48 kernel-update systemd[1]: Started Otus log timer.
```
Создаем файлик
```
     vi /var/log/log-otus
```
```
    O
    OT
    OTUS
```
Смотрим журнал
```
    journalctl -efn
```
```
    Jun 06 05:33:00 kernel-update systemd[1]: Starting Otus log service...
    Jun 06 05:33:00 kernel-update log-otus[2289]: Found "OTUS" in file /var/log/log-otus
    Jun 06 05:33:00 kernel-update systemd[1]: Started Otus log service.
```
### **Из репозитория epel установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi)**
```
    yum install -y epel-release spawn-fcgi httpd php-cli
```
Удаляем init-скрипт
```
    rm -f /etc/init.d/spawn-fcgi
```
Создаем unit-файл /etc/systemd/system/spawn-fcgi.service
```
    vi /etc/systemd/system/spawn-fcgi.service
```
```
    [Unit]
    Description=Spawn FastCGI service to be used by web servers
    After=network.target syslog.target
    
    [Service]
    Type=forking
    PIDFile=/var/run/spawn-fcgi.pid
    EnvironmentFile=/etc/sysconfig/spawn-fcgi
    ExecStart=/usr/bin/spawn-fcgi $OPTIONS
    ExecStop=/bin/kill ${MAINPID}
    
    [Install]
    WantedBy=multi-user.target
```
Раскоментим конфиг
```
    vi /etc/sysconfig/spawn-fcgi

# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
```
Запускаем
```
    systemctl daemon-reload
    systemctl enable spawn-fcgi
    systemctl start spawn-fcgi
```
Проверяем статус
```
    [root@kernel-update log]# systemctl status spawn-fcgi.service
    ● spawn-fcgi.service - Spawn FastCGI service to be used by web servers
       Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; enabled; vendor preset: disabled)
       Active: active (running) since Mon 2022-06-06 08:52:56 UTC; 12s ago
      Process: 24113 ExecStart=/usr/bin/spawn-fcgi $OPTIONS (code=exited, status=0/SUCCESS)
     Main PID: 24114 (php-cgi)
       CGroup: /system.slice/spawn-fcgi.service
               ├─24114 /usr/bin/php-cgi
               ├─24115 /usr/bin/php-cgi
               ├─24116 /usr/bin/php-cgi
               ├─24117 /usr/bin/php-cgi
               ├─24118 /usr/bin/php-cgi
               ├─24119 /usr/bin/php-cgi
               ├─24120 /usr/bin/php-cgi
               ├─24121 /usr/bin/php-cgi
               ├─24122 /usr/bin/php-cgi
               ├─24123 /usr/bin/php-cgi
               ├─24124 /usr/bin/php-cgi
               ├─24125 /usr/bin/php-cgi
               ├─24126 /usr/bin/php-cgi
               ├─24127 /usr/bin/php-cgi
               ├─24128 /usr/bin/php-cgi
               ├─24129 /usr/bin/php-cgi
               ├─24130 /usr/bin/php-cgi
               ├─24131 /usr/bin/php-cgi
               ├─24132 /usr/bin/php-cgi
               ├─24133 /usr/bin/php-cgi
               ├─24134 /usr/bin/php-cgi
               ├─24135 /usr/bin/php-cgi
               ├─24136 /usr/bin/php-cgi
               ├─24137 /usr/bin/php-cgi
               ├─24138 /usr/bin/php-cgi
               ├─24139 /usr/bin/php-cgi
               ├─24140 /usr/bin/php-cgi
               ├─24141 /usr/bin/php-cgi
               ├─24142 /usr/bin/php-cgi
               ├─24143 /usr/bin/php-cgi
               ├─24144 /usr/bin/php-cgi
               ├─24145 /usr/bin/php-cgi
               └─24146 /usr/bin/php-cgi
    
    Jun 06 08:52:56 kernel-update systemd[1]: Starting Spawn FastCGI service to be used by web servers...
    Jun 06 08:52:56 kernel-update spawn-fcgi[24113]: spawn-fcgi: child spawned successfully: PID: 24114
    Jun 06 08:52:56 kernel-update systemd[1]: Started Spawn FastCGI service to be used by web servers.
```
### **Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами**
Готовим шаблон unit-файла c %i-подстановкой
```
    vi /etc/systemd/system/httpd@.service
```
```
[Unit]
Description=The Apache HTTP Server for %i config
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
Environment=HTTPD_PID=/var/run/httpd/http-%i.pid HTTPD_CONF_DIR=/etc/httpd-otus-%i
EnvironmentFile=/etc/sysconfig/httpd-otus-%i
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND -d ${HTTPD_CONF_DIR} -c "PIDFile ${HTTPD_PID}"
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful -d ${HTTPD_CONF_DIR} -c "PIDFile ${HTTPD_PID}"
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
Клонируем конфиги
```
    cp -a /etc/sysconfig/httpd /etc/sysconfig/httpd-otus-1
    cp -a /etc/sysconfig/httpd /etc/sysconfig/httpd-otus-2
    cp -a /etc/httpd /etc/httpd-otus-1
    cp -a /etc/httpd /etc/httpd-otus-2
```
Отредактируем конфиги так, чтобы первый инстанс запускался на 8080, а второй на 8081
Запускаем сервис
```
    systemctl daemon-reload
    systemctl enable httpd@1
    systemctl start httpd@1
    systemctl enable httpd@2
    systemctl start httpd@2
```
Проверка
```
ss -nlt | grep 80
LISTEN     0      128         :::8080                    :::*
LISTEN     0      128         :::8081                    :::*
```
