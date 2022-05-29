# **Homework 6**
Переключимся на суперпользователя
```
    sudo su
```

### **Создать свой RPM (nginx)**
Устанавливаем нужное ПО и зависимости
```bash
    yum install wget rpm-build rpmdevtools gcc make
    yum install openssl-devel zlib-devel pcre-devel
```
Создадим пользака для сборки
```bash
    useradd builder -m
    su - builder
```
Создадим структуру каталогов для сборки
```
    rpmdev-setuptree
```
```
    [builder@kernel-update ~]$ ll
    total 0
    drwxrwxr-x. 7 builder builder 72 May 29 12:33 rpmbuild
    [builder@kernel-update ~]$ cd rpmbuild/
    [builder@kernel-update rpmbuild]$ ll
    total 0
    drwxrwxr-x. 2 builder builder 6 May 29 12:33 BUILD
    drwxrwxr-x. 2 builder builder 6 May 29 12:33 RPMS
    drwxrwxr-x. 2 builder builder 6 May 29 12:33 SOURCES
    drwxrwxr-x. 2 builder builder 6 May 29 12:33 SPECS
    drwxrwxr-x. 2 builder builder 6 May 29 12:33 SRPMS
```
Загружаем исходник и установим скачанный исходник командой
```
    wget https://nginx.org/packages/mainline/centos/7/SRPMS/nginx-1.19.3-1.el7.ngx.src.rpm
    rpm -Uvh nginx-1.19.3-1.el7.ngx.src.rpmсв
```
Идем в sources и изменим nginx.conf - включим режим сжатия по умолчанию (раскоментим gzip on)
```
    vi ~/rpmbuild/SOURCES/nginx.conf
```
```
    ...
        keepalive_timeout  65;
    
        :wegzip  on;
    
        include /etc/nginx/conf.d/*.conf;i
    ...
```
Собирем RPM-пакет
```
    rpmbuild -bb rpmbuild/SPECS/nginx.spec
```
Результат
```
    nginx-1.19.3-1.el7.ngx.x86_64.rpm
```
Скопируем пакет на локальную машину, пересоздадим машину из Vagrant файла и установим наш пакет
Установим плагин vagrant-scp и скопируем в другой консоли
```
    vagrant plugin install vagrant-scp
    vagrant scp e421bdc:/tmp/nginx-1.19.3-1.el7.ngx.x86_64.rpm nginx-1.19.3-1.el7.ngx.x86_64.rpm
```
Устанавливаем пакет
```
    rpm -Uvh nginx-1.19.3-1.el7.ngx.x86_64.rpm
```
Проверяем nginx.conf в системе
```
    vi /etc/nginx/nginx.conf
```
Видим, что опция gzip включена - та настройка, которую мы активировали в нашем RPM:
```
    ...
        #tcp_nopush     on;
    
        keepalive_timeout  65;
    
        gzip  on;
    
        include /etc/nginx/conf.d/*.conf;
    }
    ...
```
### **Создать свой repo и разместить там RPM**
Установим утилиту createrepo
```
    yum install createrepo
```
Создаем репозиторий в директории, где лежит rpm файл
```
     createrepo .
```
Видим созданный каталог repodata. Создание репозитория завершено
```
    -rw-r--r--. 1 vagrant vagrant 802220 May 29 12:59 nginx-1.19.3-1.el7.ngx.x86_64.rpm
    drwxr-xr-x. 2 root    root      4096 May 29 13:44 repodata
```
