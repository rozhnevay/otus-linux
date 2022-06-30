# **Homework 11**

### **Создадим inventory файл в ini-формате**
```shell script
    cat <<"EOF" > staging/hosts
[web]
nginx ansible_host=172.30.112.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=/mnt/c/Users/User/otus-linux/hw11/.vagrant/machines/nginx/virtualbox/private_key
EOF
```
Убедимся, что хост доступен через Ansible
```shell script
    ansible nginx -i staging/hosts -m ping
    nginx | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/bin/python"
        },
        "changed": false,
        "ping": "pong"
    }
```
Создадим ansible.cfg, чтобы не указывать каждый раз inventory-файл
```shell script
    cat <<"EOF" > ansible.cfg
[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
EOF
```
Теперь можно убрать инфу о пользаке из inventory (т.к. мы ее уже добавили в ansible.cfg)
```shell script
    cat <<"EOF" > staging/hosts
[web]
nginx ansible_host=172.30.112.1 ansible_port=2222 ansible_private_key_file=/mnt/c/Users/User/otus-linux/hw11/.vagrant/machines/nginx/virtualbox/private_key
EOF
```
Убедимся, что хост стал доступен без указания inventory-файла
```shell script
    ansible nginx -m ping
```
```
    nginx | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/bin/python"
        },
        "changed": false,
        "ping": "pong"
    }
```
## Ad-hoc команды
Какое ядро установлено?
```shell script
    root@DESKTOP-5P5E6L6:/mnt/c/Users/User/otus-linux/hw11# ansible nginx -m command -a "uname -r"
    nginx | CHANGED | rc=0 >>
    3.10.0-1127.el7.x86_64
```
Статус сервиса firewalld
```
    root@DESKTOP-5P5E6L6:/mnt/c/Users/User/otus-linux/hw11# ansible nginx -m systemd -a name=firewalld
    nginx | SUCCESS => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/bin/python"
        },
        "changed": false,
        "name": "firewalld",
        "status": {
            "ActiveEnterTimestampMonotonic": "0",
            "ActiveExitTimestampMonotonic": "0",
            "ActiveState": "inactive",
            "After": "polkit.service dbus.service basic.target system.slice",
            "AllowIsolate": "no",
            "AmbientCapabilities": "0",
            "AssertResult": "no",
```
Установим пакет epel-release
```shell script
    ansible nginx -m yum -a "name=epel-release state=present" -b
```
```
    nginx | CHANGED => {
        "ansible_facts": {
            "discovered_interpreter_python": "/usr/bin/python"
        },
        "changed": true,
        "changes": {
            "installed": [
                "epel-release"
            ]
        },
```
Создадим простой playbook epel.yml
```shell script
    cat <<"EOF" > epel.yml
---
- name: Install EPEL Repo
  hosts: nginx
  become: true
  tasks:
    - name: Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
EOF
```
Запустим его
```shell script
    ansible-playbook epel.yml
```
```shell script
PLAY [Install EPEL Repo] ***********************************************************************************************

TASK [Gathering Facts] *************************************************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standart repo] ********************************************************************
ok: [nginx]

PLAY RECAP *************************************************************************************************************
nginx                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Уберем репозиторий и запустим playbook снова
```shell script
    ansible nginx -m yum -a "name=epel-release state=absent" -b
    ansible-playbook epel.yml
```
Видим, что таска на install теперь в статусе changed:
```
PLAY [Install EPEL Repo] ***********************************************************************************************

TASK [Gathering Facts] *************************************************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standart repo] ********************************************************************
changed: [nginx]

PLAY RECAP *************************************************************************************************************
nginx                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Добавим установку NGINX в файл nginx.yml
```shell script
    cat <<"EOF" > nginx.yml
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  
  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages
EOF
```
Выведем в консоль все теги
```shell script
    ansible-playbook nginx.yml --list-tags
```
```shell script
    root@DESKTOP-5P5E6L6:/mnt/c/Users/User/otus-linux/hw11# ansible-playbook nginx.yml --list-tags
    
    playbook: nginx.yml
    
      play #1 (nginx): NGINX | Install and configure NGINX  TAGS: []
          TASK TAGS: [epel-package, nginx-package, packages]
```
Запустим установку только NGINX
```shell script
    ansible-playbook nginx.yml -t nginx-package
```
```shell script
    TASK [NGINX | Install NGINX package from EPEL Repo] ********************************************************************
    changed: [nginx]
    
    PLAY RECAP *************************************************************************************************************
    nginx                      : ok=2    changed=1    unreachable=0    fa
```
Добавим копирование конфига nginx в playbook.
```shell script
    cat <<"EOF" > nginx.yml
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      tags:
        - nginx-configuration
EOF
```
Также создадим шаблон для nginx.conf в формате Jinja2
```shell script
    cat <<"EOF" > templates/nginx.conf.j2    
# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}
EOF
```
Создадим handelr-ы - один на перезапуск nginx в случае обновления (соответствующий notify добавим в task yum) и handler на reload (notify на обновление шаблона).
Итоговый playbook:
```shell script
    cat <<"EOF" > nginx.yml
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes
    
    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
EOF
```
Запускаем playbook
```shell script
    ansible-playbook nginx.yml
```
```shell script
    PLAY [NGINX | Install and configure NGINX] *****************************************************************************
    
    TASK [Gathering Facts] *************************************************************************************************
    ok: [nginx]
    
    TASK [NGINX | Install EPEL Repo package from standart repo] ************************************************************
    ok: [nginx]
    
    TASK [NGINX | Install NGINX package from EPEL Repo] ********************************************************************
    ok: [nginx]
    
    TASK [NGINX | Create NGINX config file from template] ******************************************************************
    changed: [nginx]
    
    RUNNING HANDLER [reload nginx] *****************************************************************************************
    changed: [nginx]
    
    PLAY RECAP *************************************************************************************************************
    nginx                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
````
Убедимся, что сайт доступен, дёрнув curl
```shell script
    curl http://172.30.112.1:8080
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

        body {
        border: 10px solid #fff;
        margin:0;
        padding:0;
        background: #fff;
        }

        /* Links */

        a:link { border-bottom: 1px dotted #ccc; text-decoration: none; color: #204d92; }
        a:hover { border-bottom:1px dotted #ccc; text-decoration: underline; color: green; }
        a:active {  border-bottom:1px dotted #ccc; text-decoration: underline; color: #204d92; }
        a:visited { border-bottom:1px dotted #ccc; text-decoration: none; color: #204d92; }
        a:visited:hover { border-bottom:1px dotted #ccc; text-decoration: underline; color: green; }

        .logo a:link,
        .logo a:hover,
        .logo a:visited { border-bottom: none; }

        .mainlinks a:link { border-bottom: 1px dotted #ddd; text-decoration: none; color: #eee; }
        .mainlinks a:hover { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }
        .mainlinks a:active { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }
        .mainlinks a:visited { border-bottom:1px dotted #ddd; text-decoration: none; color: white; }
        .mainlinks a:visited:hover { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }

        /* User interface styles */

        #header {
        margin:0;
        padding: 0.5em;
        background: #204D8C url(img/header-background.png);
        text-align: left;
        }

        .logo {
        padding: 0;
        /* For text only logo */
        font-size: 1.4em;
        line-height: 1em;
        font-weight: bold;
        }

        .logo img {
        vertical-align: middle;
        padding-right: 1em;
        }

        .logo a {
        color: #fff;
        text-decoration: none;
        }

        p {
        line-height:1.5em;
        }

        h1 {
                margin-bottom: 0;
                line-height: 1.9em; }
        h2 {
                margin-top: 0;
                line-height: 1.7em; }

        #content {
        clear:both;
        padding-left: 30px;
        padding-right: 30px;
        padding-bottom: 30px;
        border-bottom: 5px solid #eee;
        }

    .mainlinks {
        float: right;
        margin-top: 0.5em;
        text-align: right;
    }

    ul.mainlinks > li {
    border-right: 1px dotted #ddd;
    padding-right: 10px;
    padding-left: 10px;
    display: inline;
    list-style: none;
    }

    ul.mainlinks > li.last,
    ul.mainlinks > li.first {
    border-right: none;
    }

  </style>

</head>

<body>

<div id="header">

    <ul class="mainlinks">
        <li> <a href="http://www.centos.org/">Home</a> </li>
        <li> <a href="http://wiki.centos.org/">Wiki</a> </li>
        <li> <a href="http://wiki.centos.org/GettingHelp/ListInfo">Mailing Lists</a></li>
        <li> <a href="http://www.centos.org/download/mirrors/">Mirror List</a></li>
        <li> <a href="http://wiki.centos.org/irc">IRC</a></li>
        <li> <a href="https://www.centos.org/forums/">Forums</a></li>
        <li> <a href="http://bugs.centos.org/">Bugs</a> </li>
        <li class="last"> <a href="http://wiki.centos.org/Donate">Donate</a></li>
    </ul>

        <div class="logo">
                <a href="http://www.centos.org/"><img src="img/centos-logo.png" border="0"></a>
        </div>

</div>

<div id="content">

        <h1>Welcome to CentOS</h1>

        <h2>The Community ENTerprise Operating System</h2>

        <p><a href="http://www.centos.org/">CentOS</a> is an Enterprise-class Linux Distribution derived from sources freely provided
to the public by Red Hat, Inc. for Red Hat Enterprise Linux.  CentOS conforms fully with the upstream vendors
redistribution policy and aims to be functionally compatible. (CentOS mainly changes packages to remove upstream vendor
branding and artwork.)</p>

        <p>CentOS is developed by a small but growing team of core
developers.&nbsp; In turn the core developers are supported by an active user community
including system administrators, network administrators, enterprise users, managers, core Linux contributors and Linux enthusiasts from around the world.</p>

        <p>CentOS has numerous advantages including: an active and growing user community, quickly rebuilt, tested, and QA'ed errata packages, an extensive <a href="http://www.centos.org/download/mirrors/">mirror network</a>, developers who are contactable and responsive, Special Interest Groups (<a href="http://wiki.centos.org/SpecialInterestGroup/">SIGs</a>) to add functionality to the core CentOS distribution, and multiple community support avenues including a <a href="http://wiki.centos.org/">wiki</a>, <a
href="http://wiki.centos.org/irc">IRC Chat</a>, <a href="http://wiki.centos.org/GettingHelp/ListInfo">Email Lists</a>, <a href="https://www.centos.org/forums/">Forums</a>, <a href="http://bugs.centos.org/">Bugs Database</a>, and an <a
href="http://wiki.centos.org/FAQ/">FAQ</a>.</p>

        </div>

</div>


</body>
</html>
```
