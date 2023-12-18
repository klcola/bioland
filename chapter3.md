# 安装配置计算节点操作系统镜像

**请注意，本指南仅针对 Ubuntu 22.04 server 版本有效**

## 1. 使用 debootstrap 安装 ubuntu 基本系统到 /srv/bioland/jammy 目录下
```bash
apt install debootstrap
debootstrap --arch amd64 jammy /srv/bioland/jammy

# 需清空 /srv/bioland/jammy/etc/hostname 中的主机名信息
# 否则无盘启动的节点名称会与本地服务器相同，从而造成在 /etc/hosts 中配置的主机名不生效
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

### 1）下载 ubuntu jammy 源码文件  
有许多不同的方法可以获得内核源代码，如果你已经安装了Ubuntu的一个版本，并且你想对系统上安装的内核进行更改，请使用apt-get方法（如下所述）来获取源代码。
参考链接：[BuildYourOwnKernel](https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel)
```bash
# 安装下载 linux 内核源码需要的 package
apt install dpkg-dev debhelper gawk

# 安装通用内核，主要是待会需要用这个内核的 config 文件，在这个 config 文件的基础上来定制编译我们的内核
apt install linux-image-generic
cd /root
apt-get source linux-image-unsigned-$(uname -r)

# 进入内核文件夹
cd /root/linux-5.15.0
chmod a+x debian/rules
chmod a+x debian/scripts/*
chmod a+x debian/scripts/misc/*
LANG=C fakeroot debian/rules clean
```

### 2）安装编译内核所需要的工具软件包
```bash
apt install gcc g++ dwarves libncurses5-dev flex bison libssl-dev libncurses-dev libelf-dev libpci-dev python3-dev libcap-dev bc rsync
```

### 3）配置内核
主要是在默认选项的基础上增加 nfs 文件系统相关支持及机器网卡驱动支持，<font color="red">并且这些支持一定需要直接编入内核，而不是以 module 形式存在</font>。
```bash
cp /boot/config-5.15.0-78-generic .config
make menuconfig
```
然后在相关选项内查找和 NFS root 以及 DHCP 远程启动相关的选项

Networking support --> Network options --> IP: kernel level autoconfiguration 这个选项要打开，就能看到 IP: DHCP support 和 IP: BOOTP support 等，这些也都选 y

File Systems --> Network File Systems 里把 NFS client 选 y，能看到 Root file system on NFS，这个一定要选中

配置完内核退出后，查看一下相关选项是否选中。可以用如下命令检查：

```bash
grep CONFIG_NETWORK_FILESYSTEMS .config
```
需要重点检查的内容
```
-->NFS client
CONFIG_NETWORK_FILESYSTEMS=y
CONFIG_NFS_FS=y
CONFIG_NFS_V3=y

-->NFS root
CONFIG_ROOT_NFS=y

-->Kernel level network autoconfiguration
CONFIG_IP_PNP=y
CONFIG_IP_PNP_DHCP=y
CONFIG_IP_PNP_BOOTP=y
```

另外需要注意，服务器网卡的驱动也必须编译在内核中，不能编译成 module 的形式。

### 4）编译内核
运行如下命令编译内核
```bash
# 编译会在上一级目录生成几个个deb文件，需要安装的是 linux-headers、linux-image 和 linux-libc-dev
make -j $(nproc) bindeb-pkg
```

### 5）安装内核
```
cd /root
dpkg -i linux-headers-5.15.126_5.15.126-1_arm64.deb linux-image-5.15.126_5.15.126-1_arm64.deb linux-libc-dev_5.15.126-1_arm64.deb
```
(可以根据实际情况在命令行中敲入正确的 .deb 文件名)

### 6）生成支持 nfs netboot 的 initrd 文件

首先更改 /etc/initramfs-tools/initramfs.conf 配置文件，内容如下
```
#
# initramfs.conf
# Configuration file for mkinitramfs(8). See initramfs.conf(5).
#
# Note that configuration options from this file can be overridden
# by config files in the /etc/initramfs-tools/conf.d directory.
#

#
# BOOT: [ local | nfs ]
#
# local - Boot off of local media (harddrive, USB stick).
#
# nfs - Boot using an NFS drive as the root of the drive.
#

BOOT=nfs

#
# MODULES: [ most | netboot | dep | list ]
#
# most - Add most filesystem and all harddrive drivers.
#
# dep - Try and guess which modules to load.
#
# netboot - Add the base modules, network modules, but skip block devices.
#
# list - Only include modules from the 'additional modules' list
#

MODULES=netboot

#
# BUSYBOX: [ y | n | auto ]
#
# Use busybox shell and utilities.  If set to n, klibc utilities will be used.
# If set to auto (or unset), busybox will be used if installed and klibc will
# be used otherwise.
#

BUSYBOX=auto

#
# COMPRESS: [ gzip | bzip2 | lz4 | lzma | lzop | xz | zstd ]
#

COMPRESS=zstd

#
# DEVICE: ...
#
# Specify a specific network interface, like eth0
# Overridden by optional ip= or BOOTIF= bootarg
#

DEVICE=

#
# NFSROOT: [ auto | HOST:MOUNT ]
#

NFSROOT=auto

#
# RUNSIZE: ...
#
# The size of the /run tmpfs mount point, like 256M or 10%
# Overridden by optional initramfs.runsize= bootarg
#

RUNSIZE=10%

#
# FSTYPE: ...
#
# The filesystem type(s) to support, or "auto" to use the current root
# filesystem type
#

FSTYPE=auto
```
<font color="red">注意，如果有 /etc/initramfs-tools/conf.d/driver-policy 文件，该文件中的设置会覆盖 MODULES=netboot 选项</font>

创建 initrd 文件，注意修改 initrd.img- 后面的版本号以符合实际情况
```bash
# 可修改 /etc/initramfs-tools/modules 增加对驱动模块的支持
# 比如 Intel® Ethernet Controller X710 的 iavf 模块
mkinitramfs -k -o initrd.img-5.15.126 5.15.126
mv initrd.img-5.15.126 /boot/
```
mkinitramfs 用法说明可参见 [http://manpages.ubuntu.com/manpages/bionic/man8/mkinitramfs.8.html](http://manpages.ubuntu.com/manpages/bionic/man8/mkinitramfs.8.html)

### 7）为系统设置 Linux 源文件目录
<font color="red">请注意 /lib/modules/5.15.126 中，有两个软连接，需要指向刚刚编译内核的 source code 目录 (5.15.126可根据实际版本替换)</font>
```bash
# 拷贝source code至chroot内
mv /root/linux-5.15.0 /usr/src

cd /lib/modules/5.15.126
ln -s /usr/src/linux-5.15.0 source
```

## 8. 退出 chroot 系统，将编译好的内核 copy 至 tftp 目录
```
cp /srv/bioland/jammy/boot/vmlinuz-5.15.126 /srv/tftp/vmlinuz-5.15.126-amd64
cp /srv/bioland/jammy/boot/initrd.img-5.15.126 /srv/tftp/initrd.img-5.15.126-amd64
```

## 9. 启动计算节点，选择 uefi 方式从网卡启动
如果一切正常，应该可以顺利启动到我们配置好的操作系统上

## 10. 登录计算节点，安装 lustre 文件系统
编译 lustre 文件系统客户端需要安装一些必要的依赖库软件包
```bash
# 安装编译 lustre 客户端程序需要的软件包（可能会根据实际运行环境有所变化）
apt update
apt install dpatch libjson-c-dev libkeyutils-dev libmount-dev libnl-genl-3-dev libkrb5-dev libreadline-dev libsnmp-dev libssl-dev libyaml-dev linux-headers-generic module-assistant mpi-default-dev python2 python3-dev

# 安装 lustre 源文件包
dpkg -i lustre-source_2.15.3-1_all.deb

# 查看 lustre 源文件安装路径
dpkg -L lustre-source

# 解压缩源文件包
bunzip2 -c /usr/src/lustre-3bb9e28884.tar.bz2 | tar xf -

# 编译和安装 lustre 客户端，注意这里的 --with-linux= 选项使用我们自己配置的 Linux 内核源代码目录
cd ./modules/lustre
./configure --with-linux=/usr/src/linux-5.15.0 --disable-server --with-o2ib=/usr/src/ofa_kernel/default
make debs
cd ./debs
dpkg -i lustre-client-modules-5.15.0-89-generic_2.15.3-1_arm64.deb lustre-client-utils_2.15.3-1_arm64.deb lustre-iokit_2.15.3-1_arm64.deb lustre-dev_2.15.3-1_arm64.deb
```

挂载 lustre 文件系统需要配置 /etc/modprobe.d/lustre.conf 文件，内容如下
```
# 注意，o2ib(ib0,ib1) 需要根据 ib 卡适配器的实际情况做调整，比如系统中有 4 张 ib 卡，
# 设备名分别为 ib0,ib1,ib2,ib3，则应写成 o2ib(ib0,ib1,ib2,ib3)
options lnet networks="o2ib(ib0,ib1)"
```
