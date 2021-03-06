# **Homework 7**
Переключимся на суперпользователя
```
    sudo su
```

### **Сменить пароль root**
0 - Зайдем в BIOS при загрузке (F2)
1 - Отредактируем параметры загрузки в скрипте BIOS - добавим rd.break, продолжим загрузку (Ctrl+X)
2 - Попадаем в emergency mode. Сменим пароль root
Перемонтируем sysroot в режиме rw
```
    mount -o remount,rw /sysroot   
```
Переходим в корневую ФС нашей систем
```
    chroot /sysroot
```
Устанавливаем новый пароль
```
    passwd
```
Создадим relabel файл и восстановим ro на файловой системе. Reboot
```
    touch ./autorelabel
    exit
    mount -o remount,rw /sysroot
    reboot
```
### **Установить систему с LVM, после чего переименовать VG** 
Подготовим раздел

    pvcreate /dev/sdb
    vgcreate vg /dev/sdb
    lvcreate -n lv -l +100%FREE /dev/vg

Примонтируем, создадим файл
```
    mkfs.xfs /dev/vg_root/lv_root
    mount /dev/vg/lv /mnt
    touch mnt/file
```
Переименуем VG
```
    [root@otuslinux mnt]# vgrename vg vg_renamed
     Volume group "vg" successfully renamed to "vg_renamed"
```
Проверяем файл
```
    [root@otuslinux mnt]# ll
    total 0
    -rw-r--r--. 1 root root 0 Jun  3 07:07 file
```
Файл на месте, VG переименована
```
    [root@otuslinux mnt]# vgs
      VG         #PV #LV #SN Attr   VSize   VFree
      vg_renamed   1   1   0 wz--n- 248.00m    0
```
### **Добавить модуль в initrd** 
Добавляем модуль
```
    mkdir /usr/lib/dracut/modules.d/01test
    cat >module-setup.sh<<EOF
```
```
    #!/bin/bash
    check() {
        return 0
    }
    depends() {
        return 0
    }
    install() {
        inst_hook cleanup 00 "${moddir}/test.sh"
    }
```
```
    chmod +x module-setup.sh
```
Добавляем test.sh
```
    #!/bin/bash
    exec 0<>/dev/console 1<>/dev/console
    2<>/dev/console
    cat <<'msgend'
    Hello! You are in dracut module!
    ___________________
    < I'm dracut module from otus >
    --------------------
    \
     .-----.
     | o_o |
     | \_/ |
     / / \ \
     ( | | )
    / '\_ _/`\
    \ ) ( /
```
```
    chmod +x test.sh
```
Пересобираем наш новый модуль
```
    mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
```
Убедимся, что наш модуль на месте
```
    lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
```
```
    [root@boot 01test]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
    test
```
Перезагружаемся и смотрим файл с логом загрузки
```
    less -R /var/log/boot.log
```
Видим, что все ок, картинка на месте:
```
    [  OK  ] Reached target Initrd Default Target.
             Starting dracut pre-pivot and cleanup hook...
    [    4.908491] dracut-pre-pivot[429]: //lib/dracut/hooks/cleanup/00-test.sh: line 16: warning: here-document at line 4 delimited by end-of-file (wanted `msgend')
        Hello! You are in dracut module!
        ___________________
        < I'm dracut module from otus >
        --------------------
        \
         .-----.
         | o_o |
         | \_/ |
         / / \ \
         ( | | )
        / '\_ _/`\
        \ ) ( /
```

