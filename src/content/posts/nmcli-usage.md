---
title: nmcli-usage
published: 2024-09-13
description: 'my frequently used cmds of nmcli'
image: ''
tags: ["linux", "network"]
category: cmd
draft: false 
lang: ''
---
# nmcli相关命令

## section
nmcli总共有八大部分的参数
| section | 描述 |
|:--------|:-----|
| help | 获取帮助 |
| general | 获得网络管理器相关的状态和全局参数的值 |
| networking | 开启、重启、停止网络管理器 |
| radio | 管理无线设备和相关协议 |
| connection | 管理连接 | 
| device | 管理网络设备 | 
| agent | 配置和管理安全设置 |
| monitor | 监控网络的相关变化情况 |

## dev
dev管理的是机器的硬件设备
#### 检查设备联网情况
```shell
nmcli dev status
```
以上命令会打印出当前机器上所有可能用来联网的设备以及它是否有网络连接。
```shell
cassius@ubuntu:~$ nmcli dev status
DEVICE         TYPE      STATE         CONNECTION
eth0           ethernet  connected     Wired connection 1
wlan0          wifi      connected     AirHustA
docker0        bridge    connected     docker0
p2p-dev-wlan0  wifi-p2p  disconnected  --
l4tbr0         bridge    unmanaged     --
dummy0         dummy     unmanaged     --
rndis0         ethernet  unmanaged     --
usb0           ethernet  unmanaged     --
lo             loopback  unmanaged     --
```
例如，在我的设备上显示以太网和无线网都有连接，并且docker通过桥接模式连接到宿主机。

#### 查看可用的wifi
```shell
nmcli dev wifi list
```
以上命令就会打印出设备检测到的wifi。同一个wifi可能会被打印出来多次，因为wifi使用着不同的信道。

## con
con是管理设备的相关连接配置，如定义IP地址、子网掩码、网关、DNS服务器等
#### 查看联网状态
如果只希望查看连了网的设备，可以通过以下命令查看
```shell
nmcli con show
```
查看结果如下
```shell
cassius@ubuntu:~$ nmcli con show
NAME                UUID                                  TYPE      DEVICE
Wired connection 1  30b11a44-5fd9-3717-9b55-71879f7d85b2  ethernet  eth0
AirHustA            ba28574e-ab8a-47bc-a0a2-631e0e1ee0ca  wifi      wlan0
docker0             905202de-05c7-45a3-8d96-0d83022c3bcd  bridge    docker0
```
更进一步的，可以查看连接中的设备的更详细的信息，只要在show后面加上具体的设备名称，如在我的情况下，就应该是
```shell
nmcli con show 'Wired connection 1'
```
这里之所以需要加单引号是因为设备名称中有空格，如果没有空格这里就不需要加单引号了。

#### 添加新的连接配置文件
```shell
 nmcli con add con-name <connection-name> type wifi ifname wlan0 ssid <wifi-name> ipv4.method auto
```
通过以上命令，可以创建一个新的连接，通过wifi俩姐到一个叫做`<connection-name>`的wifi，并且指定ipv4地址通过DHCP方法获取。一般情况下，该指令需要sudo权限。

虽然一般情况下连接的是wifi，但是如果要连接的是以太网的话，`type`后面跟的应该是`ethernet`， `ifname`后面也应该跟着以太网口的设备名，一般是`eth0`

##### 输入wifi密码
如果wifi是使用wpa2协议来保护的话，后续通过以下两条命令把wifi密码添加到配置文件中。
```shell
nmcli connection modify CS_INF wifi-sec.key-mgmt wpa-psk
nmcli connection modify CS_INF wifi-sec.psk <wifi-password>
```

##### 开启热点
热点本质上也只是一个本机的连接，所以也是通过nmcli来添加配置文件，就可以连接
```shell
nmcli con add type wifi ifname <wireless-dev-name> con-name <connection-name> ssid <hot-spot-name> [autoconnect yes]
```
以上命令中，`<wireless-dev-name>`一般是诸如`wlp3s0`这样的设备名，注意不可能是`wifi`

#### 修改连接
```shell
nmcli con modify <connection-name> ipv4.method manual ipv4.address 10.0.0.10/8 ipv4.gateway 10.10.10.1
```
以上命令就相当于在系统设置中指定ip地址为手动模式，设置了ipv4地址为10.0.0.10，子网掩码为255.0.0.0，网关为10.10.10.1

##### 自动连接
如果希望设备每次能够自动尝试连接上某一个连接，可以使用
```shell
nmcli con modify <conection-name> autoconnect yes
```
来要求设备每次都自动尝试连接到`<connection-name>`上

#### 删除连接
```shell
nmcli con del <connection-name>
```

#### 断开连接
```shell
nmcli con down <connection-name>
```

#### 连接
```shell
nmcli con up <connection-name>
```