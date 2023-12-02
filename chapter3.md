# 安装配置计算节点操作系统镜像

**请注意，本指南仅针对 Ubuntu 22.04 server 版本有效**

## 1. 使用 debootstrap 安装 ubuntu 基本系统到 /srv/bioland/jammy 目录下
```bash
apt install debootstrap
debootstrap --arch amd64 jammy /srv/bioland/jammy

# 需清空 /srv/nfs4/jammy/etc/hostname 中的主机名信息
# 否则无盘启动的节点名称会与本地服务器相同，在 /etc/hosts 中配置的主机名不生效
```

## 2. 配置 nfs 基本系统的 apt 源
```bash
cp /etc/apt/sources.list /srv/bioland/jammy/etc/apt/sources.list
```

## 3. 挂载 dev 和 sys 文件系统
```bash
mount -o bind /dev /srv/bioland/jammy/dev
mount -o bind /sys /srv/bioland/jammy/sys
```

## 4. 配置 nfs 基本系统，chroot 切换至该系统，并设置相关环境变量
```bash
LANG=C.UTF-8 chroot /srv/bioland/jammy /bin/bash
export PS1="(BL)$PS1"
```

<font color="red">注意以下步骤均在 chroot 系统内运行</font>
## 5. (chroot内)配置计算节点操作系统
```bash
apt update
mount none /proc -t proc

# 升级软件到当前最新稳定版本
apt upgrade

# 推荐安装的软件（非必须，但挺有用）
apt install vim vim-syntastic
```

## 6. (chroot内)安装 sshd、fail2ban 等服务
```bash
apt install openssh-server fail2ban

# fail2ban 的配置请参考 第二章 的相关内容
```

## 7. (chroot内)编译内核，以支持通过网卡启动系统

### 下载 ubuntu jammy 源码文件  
有许多不同的方法可以获得内核源代码，如果你已经安装了Ubuntu的一个版本，并且你想对系统上安装的内核进行更改，请使用apt-get方法（如下所述）来获取源代码。
参考链接：[BuildYourOwnKernel](https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel)
```bash
# 安装下载 linux 内核源码需要的 package
apt install dpkg-dev debhelper gawk

# 安装通用内核，主要是待会需要用这个内核的 config 文件，在这个 config 文件的基础上来定制编译我们的内核
apt install linux-image-generic

apt-get source linux-image-unsigned-$(uname -r)

# 进入内核文件夹
cd linux-5.15.0
chmod a+x debian/rules
chmod a+x debian/scripts/*
chmod a+x debian/scripts/misc/*
LANG=C fakeroot debian/rules clean
```

### 安装编译内核所需要的工具软件包
```bash
apt install gcc g++ dwarves libncurses5-dev flex bison libssl-dev libncurses-dev libelf-dev libpci-dev python3-dev libcap-dev bc rsync
```

### 配置内核
主要是在默认选项的基础上增加 nfs 文件系统相关支持及机器网卡驱动支持，<font color="red">并且这些支持一定需要直接编入内核，而不是以 module 形式存在</font>。
```bash
cp /boot/config-5.15.0-78-generic .config
make menuconfig
```
