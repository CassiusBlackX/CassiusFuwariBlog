---
title: docker-install
published: 2024-04-02
description: 'how I manage to install docker'
image: ''
tags: ["linux", "docker"]
category: deployment
draft: false 
lang: ''
---
# docker
## docker pull走自己的代理
### 确保/创建目录
```shell
sudo mkdir -p /etc/systemd/system/docker.service.d
```
### 创建代理配置文件
```shell
sudo vim /etc/systemd/system/docker.service.d/proxy.conf
```
将以下内容添加到`proxy.conf`文件中
```toml
[Service]
Environment="HTTP_PROXY=http://localhost:7890"
Environment="HTTPS_PROXY=http://localhost:7890"
```
### 重启Docker服务
```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```
### 检测是否成功
使用命令
```shell
docker info | grep HTTP
```
如果能看到设置的两条代理，说明成功。