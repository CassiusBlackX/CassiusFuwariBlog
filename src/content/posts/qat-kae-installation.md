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


# 在服务器上安装qat
在戴尔PowerEdge R750xs上，安装DH895XCC对应的QAT sriov的驱动
## 系统准备
### 依赖安装
```shell
sudo apt-get update
sudo apt-get install -y libsystemd-dev
sudo apt-get install -y libudev-dev
sudo apt-get install -y libreadline6-dev
sudo apt-get install -y pkg-config
sudo apt-get install -y libxml2-dev
sudo apt-get install -y libpci-dev
sudo apt-get install -y libboost-all-dev
sudo apt-get install -y libelf-dev
sudo apt-get install -y linux-headers-$(uname -r)
sudo apt-get install -y build-essential
sudo apt-get install -y nasm
sudo apt-get install -y zlib1g-dev
sudo apt-get install -y libssl-dev
sudo apt-get install -y libnl-3-dev libnl-genl-3-dev
sudo apt-get install -y gcc-12
```
需要安装以上依赖包

### 驱动安装
手动下载QAT的tar包，上传到服务器上，根据intel官网的手册照着做就可以了。
```shell
export ICP_ROOT=/QAT
mkdir -p $ICP_ROOT
cd $ICP_ROOT
# mv QAT tar to current directory
tar -zxof QAT20.L.*.tar.gz
# ./configure # defualt
./configure --enable-icp-sriov=host
make -j 48 install
make samples-install
usermod -a -G qat `whoami`
# exit and re-login
lsmod | grep qat  # should see a series of ouput after the cmd
service start qat_service
systemctl status qat_service # check the status of qat_service
```

### 开启系统IOMMU
修改`/etc/default/grub`文件中的内容,把`GRUB_CMDLINE_LINUX_DEFAULT`的内容修改为如下形式
```plaintext
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=pt intel_iommu=on"
```
也可选的直接在这里开启巨页
```plaintext
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash default_hugepagesz=1G hugepagesz=1G hugepages=30 iommu=pt intel_iommu=on"
```
然后更新grub
```shell
sudo update-grub
sudo reboot
```
重启以后才生效

## 准备测试环境
### dpdk
克隆下来dpdk的代码，并且编译
```shell
meson -Dexamples=all setup build
cd build
ninja
sudo ninja install
```
### qemu-system
安装qemu-system-x86_64
```shell
sudo apt install qemu-system
```
### 分配巨页
> 如果在之前调整`GRUB_CMDLINE_LINUX_DEFAULT`的时候就已经配置过巨页了，这一步就可以跳过
```shell
echo 8192 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```
### 启动dpdk的vhost_crypto
```shell
sudo ./build/examples/dpdk-vhost_crypto \
  -l 1-3 -n 4  -- \
  --socket-file 2,/tmp/vhost_crypto.sock
```
### 启动虚拟机
```shell
qemu-system-x86_64 \
        -cpu host \
        -m 8192 \  # 这里分配的内存大小必须和后面的-object memory-backend-file一致，受限于系统中分配的巨页总大小
        -smp 16 \  # 为了在虚拟机里面调试方便，编译速度快一点，所以多分配一些核心给虚拟机
        -enable-kvm \
        -drive file=vcrypto.qcow2,format=qcow2 \
        -nographic \
        -netdev user,id=mynet0,hostfwd=tcp::2222-:22 -device e1000,netdev=mynet0 \  # 吧虚拟机的22端口映射到本地的2222端口，方便通过ssh连接到虚拟机
        -chardev socket,id=chr0,path=/tmp/vhost_crypto.sock \
        -object cryptodev-vhost-user,id=crypto0,chardev=chr0 \
        -device virtio-crypto-pci,cryptodev=crypto0 \
        -object memory-backend-file,id=mem,size=8G,mem-path=/dev/hugepages,share=on \
        -mem-prealloc \
        -numa node,memdev=mem
```
### 进入虚拟机的准备
绑定设备
```shell
echo "ubuntu" | sudo -S modprobe uio_pci_generic
echo "ubuntu" | sudo -S sh -c "echo -n 0000:00:04.0 > /sys/bus/pci/drivers/virtio-pci/unbind"
echo "ubuntu" | sudo -S sh -c 'echo "1af4 1054" > /sys/bus/pci/drivers/uio_pci_generic/new_id'
```
在虚拟机里也挂载巨页
```shell
echo "ubuntu" | sudo -S mkdir -p /mnt/huge
echo "ubuntu" | sudo -S mount -t hugetlbfs nodev /mnt/huge
```
分配巨页
```shell
echo "ubuntu" | sudo -S sh -c "echo 2048 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages"
```

### 运行vcrypto-engine后端
### 通过openssl测试vcrypto-engine前端
使用多个进程来测试openssl在不使用和使用vcrypto engine，在同时处理多个加密任务时候的性能
```shell
#!/bin/bash

sizes=(16 64 256 1024 2048)
num_processes=50 # 并发进程数
tmp_dir=$(mktemp -d)

# 输出表头
printf "%-10s %-15s %-15s %-10s\n" "BlockSize" "DefaultSpeed" "VcryptoSpeed" "Ratio"

for size in "${sizes[@]}"; do
  default_speeds=()
  vcrypto_speeds=()

  # 并行测试默认实现
  for ((i=0; i<num_processes; i++)); do
    (
      result=$(openssl speed -elapsed -evp aes-256-cbc -bytes "$size" 2>/dev/null | grep "^aes-256-cbc")
      speed=$(awk '{print $2}' <<< "$result")
      echo "$speed" > "$tmp_dir/default_${size}_$i"
    ) &
  done
  wait

  # 并行测试vcrypto实现
  for ((i=0; i<num_processes; i++)); do
    (
      result=$(OPENSSL_ENGINES=/usr/lib/x86_64-linux-gnu/engines-1.1 openssl speed -elapsed -engine vcrypto -evp aes-256-cbc -bytes "$size" 2>/dev/null | grep "^aes-256-cbc")
      speed=$(awk '{print $2}' <<< "$result")
      echo "$speed" > "$tmp_dir/vcrypto_${size}_$i"
    ) &
  done
  wait

  # 收集结果并计算平均值
  default_avg=$(awk '{sum+=$1} END {print int(sum/NR)}' "$tmp_dir"/default_"$size"_* 2>/dev/null || echo "N/A")
  vcrypto_avg=$(awk '{sum+=$1} END {print int(sum/NR)}' "$tmp_dir"/vcrypto_"$size"_* 2>/dev/null || echo "N/A")

  # 计算加速比
  if [[ "$default_avg" != "N/A" && "$vcrypto_avg" != "N/A" && "$vcrypto_avg" -ne 0 ]]; then
    ratio=$(awk -v d="$default_avg" -v v="$vcrypto_avg" 'BEGIN { printf "%.2f", d/v }')
  else
    ratio="N/A"
  fi

  # 输出结果
  printf "%-10s %-15s %-15s %-10s\n" "${size}B" "${default_avg}k" "${vcrypto_avg}k" "$ratio"
done

# 清理临时文件
rm -rf "$tmp_dir"
```
> TODO: 如果仅使用单进程测试的情况下，不使用vcrypto engine比使用了还能高出来8万多倍！
> 多线程下能够稍微反超，但是可能没有成功用好qat

# 实验
## openssl加解密性能
### 说明
目前只测试对称加密，aes-256-cbc算法，不同的块大小
| 是否bind | vhost中是使用的设备 | 256B时候的默认性能 | 256B时候的vcrypto-engine性能 |
|:--------|:-------------------|:-----------------|:----------------------------|
|否|--vdev crypto_openssl|158075k|172273k|
|是|--vdev crypto_openssl|158793k|172479k|
|是|无|158911k|174553k|