---
title: qat-kae-installation
published: 2025-08-08
description: '安装qat和kae时候的一些踩坑记录'
image: ''
tags: ["qat", "kae"]
category: 'deployment'
draft: false 
lang: ''
---
# 在一台云主机上安装intel qat

### dependencies

+ `pkg-config` -> `sudo apt update && sudo apt install pkg-config`
+ `boost` -> `sudo apt install libboost-all-dev`
+ `udev` -> `sudo apt install libudev-dev`
+ `nasm`

### systemd issues

在ubuntu24.04上,虽然可以通过`apt`轻松的安装qat_driver和qat_lib和qat_zip,但是似乎ubuntu会把qat_service.service自动软链接到/dev/null上,导致qat_service没有办法自动启动.

因此,必须检查

```shell
ls -l /usr/lib/systemd/system/qat_service.service
```

是否是软链接到了/dev/null下,如果是的话就把这个给删除,然后使用命令

```shell
systemctl unmask qat_service.service
systemctl enable qat_service.service
systemctl start qat_service.service
```


## QATZip
### compression
#### 调用栈
`qzCompress`->`qzCompressExt`->`qzCompressCrcExt`
#### init
`qzInit`:


## kae installation
### 环境准备
#### 硬件
在Kunpeng 920 7270Z的服务器上,BIOS版本必须至少是21.17才可以看得到各种kae的加速器设备

当确定BIOS版本正确后,根据华为网站上的检查调理顺着查看,应该三种设备都可以看得到

#### 软件
OpenEuler22.03_sp3自带openssl,但是openssl,所以不需要源码安装openssl了

但是需要注意的是,openssl所在的路径将不再是`/usr/local/lib/engine-1.1/`,所以在后续设置`OPENSSL_ENGINES`环境变量之前,需要首先检查OpenSSL的这个目录所在位置,使用命令
```shell
penssl version -a | grep ENGINESDIR
```
获得的输出即为OPENSSL_ENGINES的所应当对应的值

### 安装
确保`OPENSSL_ENGINES`环境变量设置正确,其它步骤和华为网站上一致,但是不要使用rpm安装,最好使用源码编译安装,rpm里面有一些环境变量被写死了,因为openssl的路径不一致的问题,导致rpm安装后openssl还是无法使用kae的
