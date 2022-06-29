# **Homework 11**

### **Создадим inventory файл в ini-формате**
```bash
    cat <<"EOF" > staging/hosts
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=/Users/alexon/IdeaProjects/otus-linux/hw11/.vagrant/machines/nginx/vmware_desktop/private_key
EOF
```
Убедимся, что хост доступен через Ansible
```bash
    alexon@Aleksejs-MacBook-Pro hw11 % ansible nginx -i staging/hosts -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
Создадим ansible.cfg, чтобы не указывать каждый раз inventory-файл
```bash
    cat <<"EOF" > ansible.cfg
[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
EOF
```
Теперь можно убрать инфу о пользаке из inventory (т.к. мы ее уже добавили в ansible.cfg)
```bash
    cat <<"EOF" > staging/hosts
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=/Users/alexon/IdeaProjects/otus-linux/hw11/.vagrant/machines/nginx/vmware_desktop/private_key
EOF
```
Убедимся, что хост стал доступен без указания inventory-файла
```bash
    ansible nginx -m ping
```
```
    nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```
## Ad-hoc команды
Какое ядро установлено?
```
    alexon@Aleksejs-MacBook-Pro hw11 % ansible nginx -m command -a "uname -r"
nginx | CHANGED | rc=0 >>
5.4.0-89-generic
```
Статус сервиса firewalld
```
    alexon@Aleksejs-MacBook-Pro hw11 % ansible nginx -m systemd -a name=firewalld
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "name": "firewalld",
    "status": {
        "ActiveEnterTimestampMonotonic": "0",
        "ActiveExitTimestampMonotonic": "0",
        "ActiveState": "inactive",
        "AllowIsolate": "no",
        "AllowedCPUs": "",
        "AllowedMemoryNodes": "",
        "AmbientCapabilities": "",
        "AssertResult": "no",
        "AssertTimestampMonotonic": "0",
        "BlockIOAccounting": "no",
        "BlockIOWeight": "[not set]",
        "CPUAccounting": "no",
        "CPUAffinity": "",
        "CPUAffinityFromNUMA": "no",
        "CPUQuotaPerSecUSec": "infinity",
        "CPUQuotaPeriodUSec": "infinity",
        "CPUSchedulingPolicy": "0",
        "CPUSchedulingPriority": "0",
```
Установим пакет epel-release
apt_repository:
repo: deb http://repo.mongodb.org/apt/ubuntu {{ ansible_distribution_release | lower }}/mongodb-org/3.2 multiverse
state: present
```
    ansible nginx -m apt -a "repo=deb http://nginx.org/packages/ubuntu"
```