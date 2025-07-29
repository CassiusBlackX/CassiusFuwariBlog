---
title: install vm in qemu using iso files
published: 2025-07-28
description: '从一个iso文件安装一台qemu虚拟机'
image: ''
tags: ["qemu", "virtualization"]
category: 'deploy'
draft: false 
lang: ''
---
# 从iso文件安装一台qemu虚拟机
## 环境准备
`qemu`,`qemu-img`

## 创建系统镜像
```shell
qemu-img create imagename.qcow2 -f qcow2 40G # size of volume in the last parameter
```

## 无图形化界面安装
### 提取内核文件
因为在绝大多数发行版的iso文件中，安装都是依赖一个图形化界面的，所以一般可能需要使用vnc或者sdl。但是在只通过ssh连接到服务器的情况下，这两个方法都不适用，即使在qemu中加上了`-nographic`，也没有用。

下面的步骤就是为了能够不依赖图形化界面也能够安装系统的准备。
```shell
mount ubuntu-24.04.2-live-server-amd64.iso /mnt/temp -o loop
```

### 创建虚拟机
```shell
qemu-system-x86_64 \
    -m 2048 \
    -smp 2 \
    -enable-kvm \
    -drive file=ubuntu24.qcow2,format=qcow2 \
    -cdrom ./ubuntu-24.04.2-live-server-amd64.iso \
    -nographic \
    -serial mon:stdio \
    -append 'console=ttyS0,115200n8'
    --kernel {path-to-vmlinuz} \
    --initrd {path-to-initrd}
```
就可以进入到ubuntu的安装界面，并且可以通过方向键来控制光标。

系统安装完成后推出qemu，再重新进入qemu，不要带`cdrom`参数，就可以进去了。

## 通过VNC连接安装qemu虚拟机
通过以下命令启动vnc的安装qemu虚拟机
```shell
qemu-system-x86_64 \
    -m 4096 \
    -smp 4 \
    -enable-kvm \
    -drive file=openeuler.qcow2,format=qcow2 \
    -cdrom ./openEuler-22.03-LTS-SP3-x86_64-dvd.iso \
    -cpu host \
    -vnc :1 \
    -k en-us
```
其中`-vnc :1`会在服务器上的5901端口启动一个vnc服务，如果只通过ssh连接上服务器的话，最方便的方法是通过ssh的端口转发，把服务器的5901端口映射到本地，然后使用诸如tigerVNC之类的软件连接。
```shell
ssh -L 5901:localhost:5901 <username>@<server-ip> -p <ssh-port> -N -f
```
使用上面这个命令，会把服务器的5901端口(第一个5901)映射到本地的5901端口(第二个5901),然后使用tigerVNC的时候访问localhost:5901就可以了。需要注意，在访问的时候，最好保证这个ssh终端不被关闭。