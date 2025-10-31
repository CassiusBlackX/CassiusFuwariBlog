---
title: qemu-via-virsh
published: 2025-10-31
description: 使用libvirtd和virsh来配置和管理虚拟机
image: ''
tags: ['qemu']
category: deployment
draft: false 
lang: ''
---
# 安装虚拟化组件
1. 安装qemu和libvirt组件
```sh
yum install -y qemu libvirt
```
2. 启动libvirtd服务
```sh
systemctl start libvirtd
```
3. 验证
```sh
ls /dev/kvm # 有输出就正常
ls /sys/module/kvm
# parameters uevent
```

# 准备虚拟机网络
## 搭建linux网桥
1. 安装bridge-utils包
```sh
yum install -y bridge-utils
```
2. 创建网桥br0
```sh
brctl addbr br0
```
3. 将一个物理网卡绑定到网桥
```sh
brctl addif br0 eno2
```
4. 清空eno2的IP地址
```sh
yum install -y net-tools
ifconfig eno2 0.0.0.0
```
5. 配置静态IP
```sh
ifconfig br0 192.168.1.2 netmask 255.255.255.0
```
## 配置流量转发
虚拟机之后讲使用br0上网，但是仍然需要具体的流量转发。
### 启用IP转发
```sh
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
### 添加MASQUERADE规则
1. 查看zone
```sh
firewall-cmd --get-active-zones
# 不出意外，用于上网的zone是public
```
2. 永久启用masquerade
```sh
sudo firewall-cmd --permanent --zone=public --add-masquerade
sudo firewall-cmd --reload
```
3. 确保br0所在的zone被信任
```sh
sudo firewall-cmd --permanent --zone=trusted --add-interface=br0
sudo firewall-cmd --reload
```


# libvirt相关配置
## 把普通用户加入到libvirt用户组
1. 添加到组
```sh
usermod -aG libvirt $(whoami)
```
2. 配置非root用户环境变量
```sh
# ~/.bashrc
export LIBVIRT_DEFAULT_URI="qemu:///system"
```
3. source .bashrc

# virsh配置
## virsh配置文件
```xml
<domain type='kvm'>
        <name>oetest</name>
        <memory unit='GiB'>8</memory>
        <vcpu>8</vcpu>
        <cpu mode='host-passthrough'>
                <topology sockets='4' cores='2' threads='1'/>
        </cpu>
        <iothreads>2</iothreads>
        <devices>
                <disk type='file' device='cdrom'>
                        <source file='/var/lib/libvirt/images/openEuler-24.03-LTS-SP1-aarch64-dvd.iso'/>
                        <target dev='sdb' bus='scsi'/>
                        <readonly/>
                        <boot order='2'/>
                </disk>
                <disk type='file' device='disk'>
                        <driver name='qemu' type='qcow2' cache='none' io='native' iothread="1"/>
                        <source file='/var/lib/libvirt/images/openEuler-24.03-LTS-SP1-aarch64.qcow2'/>
                        <target dev='vda' bus='virtio'/>
                        <boot order='1'/>
                </disk>
                <interface type='bridge'>
                        <source bridge='br0'/>
                        <model type='virtio'/>
                </interface>
                <graphics type='vnc' listen='0.0.0.0' port='5900' keymap='en-us'/>
                <controller type='usb' model='ehci'/>
                <input type='tablet' bus='usb'/>
                <input type='keyboard' bus='usb'/>
        </devices>
        <os>
                <type arch='aarch64' machine='virt'>hvm</type>
                <loader readonly='yes' type='pflash'>/usr/share/edk2/aarch64/QEMU_EFI-pflash.raw</loader>
                <nvram>/var/lib/libvirt/qemu/nvram/openEulerVM.fd</nvram>
        </os>
        <seclabel type='dynamic' model='dac' relabel='yes'/>
</domain>
```
## virsh简单命令
|命令|含义|
|:--|:--|
|`virsh define <XML file>`|定义持久化虚拟机，定义完成后虚拟机处于关闭状态，需要start|
|`virsh create <XML file>`|创建一个临时虚拟机，创建完后处于运行状态|
|`virsh start <VM Instance>`|启动虚拟机|
|`virsh shutdown <VM Instance>`|关闭虚拟机，使用虚拟机的关机流程|
|`virsh destroy <VM Instance>`|强制关闭虚拟机|
|`virsh reboot <VM Instnace>`|重启虚拟机|
|`virsh undefine <VM Instance>`|销毁持久虚拟机，虚拟机生命周期完结，之后无法进行任何操作|

## 启动虚拟机
使用virsh create命令启动
### VNC连接到虚拟机
