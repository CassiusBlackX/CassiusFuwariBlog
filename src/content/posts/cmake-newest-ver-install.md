---
title: install newest version of cmake
published: 2024-04-02
description: 'how to install newest version of cmak'
image: ''
tags: ["linux", "cmake"]
category: deployment
draft: false 
lang: ''
---

# 在ubuntu上安装更新版本的cmake
### motivation
+ ubuntu上通过`apt`安装的cmake版本都非常的老，在构建一些项目的时候可能会无法成功。
+ `snap`有时候会突然死掉，而且无法通过`systemctl`重启，这样的情况下甚至没有办法卸载snap安装的cmake

### 安装
在Linux上，遇事不决就应该尝试源码构建，这永远是最方便的方法，而且在性能相对较可的电脑上也不会需要很多时间。

#### 获取cmake源码
先确定自己想要的cmake的版本，目前cmake大版本已经出到了4，但是我暂时用不到4的任何特性，所以我选择安装cmake3.31.3
```shell
version=3.11
build=3
mkdir cmake-repo-temp & cd cmake-repo-temp
wget https://cmake.org/files/v$version/cmake-$version.$build.tar.gz
tar -xzvf cmake-$version.$build.tar.gz
cd cmake-$version.$build/
```

#### 编译并安装cmake
```shell
./bootstrap
make -j$(nproc)
sudo make install
```

#### 测试cmake
```shell
cmake --version
```
如果看到输出版本为3.31.3，则说明安装成功！