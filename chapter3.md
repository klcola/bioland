# 安装配置计算节点操作系统镜像

## 1. 使用 debootstrap 安装 ubuntu 基本系统到 /srv/bioland/jammy 目录下
```bash
apt install debootstrap
debootstrap --arch amd64 jammy /srv/bioland/jammy

# 需清空 /srv/nfs4/jammy/etc/hostname 中的主机名信息
# 否则无盘启动的节点名称会与本地服务器相同，在 /etc/hosts 中配置的主机名不生效
```

## 1. 配置 nfs 基本系统的 apt 源
```bash
cp /etc/apt/sources.list /srv/bioland/jammy/etc/apt/sources.list
```

1. 挂载 dev 和 sys 文件系统
```bash
mount -o bind /dev /srv/bioland/jammy/dev
mount -o bind /sys /srv/bioland/jammy/sys
```

1. 配置 nfs 基本系统，chroot 切换至该系统，并设置相关环境变量
```bash
LANG=C.UTF-8 chroot /srv/bioland/jammy /bin/bash
export PS1="(n)$PS1"
```

<font color="red">注意以下步骤均在 chroot 系统内运行</font>
1. (chroot内)配置计算节点操作系统
```bash
apt update
mount none /proc -t proc
```

1. (chroot内)安装 sshd 服务
apt install openssh-server 
```

1. 
