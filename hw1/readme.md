# **Homework 1**

### **Connect repo and install needed software**
```
    sudo yum install -y http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
    sudo yum update -y
    sudo yum install yum-utils ncurses-devel make kernel-devel bc gcc openssl-devel flex bison libssl-dev pkg-config ncurses-devel libncurses-dev openssl-devel elfutils-libelf-devel perl wget -y
```

### **Download kernel sources**
```
    sudo mkdir /usr/src/kernel
    cd /usr/src/kernel
    sudo wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.5.1.tar.xz
    sudo tar xf linux-5.5.1.tar.xz
    cd linux-5.5.1
```

### **Copy config from old kernel config**
```
    sudo cp /boot/config-3.10.0-1127.el7.x86_64 /usr/src/kernel/linux-5.5.1/.config
```

### **Make kernel in parallel on 8 cores**
```
    sudo make allmodconfig
    sudo make -j 8 && make -j 8 modules
```

### **Install modules and core**
```
    sudo make -j 8 modules_install && sudo make -j 8 install
```

### **Refresh config and make new kernel load by default**
```
    sudo grub2-mkconfig -o /boot/grub2/grub.cfg
    sudo grub2-set-default 0
```
### **Reboot system**
```
    sudo reboot
```
