---
title: samba-configuration
published: 2025-04-02
description: 'how to configure samba in linux'
image: ''
tags: ["linux", "samba", "network"]
category: cmd
draft: false 
lang: ''
---
# samba配置
samba是为了用来创建共享文件夹的。可以与局域网下的其它Windows或Linux电脑共享文件夹。
```shell
sudo apt update && sudo apt upgrade
sudo apt install samba
```
编辑`smb.conf`文件，配置自己的共享文件夹
```shell
[samba_share]  # the driver name in which you input in Windows explorer
   comment = Samba Share
   path = /samba_share  # path to your samba shared directory
   browsable = yes
   guest ok = yes
   read only = no
   create mask = 0755
```
修改文件后需要重启smbd服务`sudo systemctl restart smbd`
#### 添加samba用户
在局域网内其它电脑连接samba共享文件夹的时候，需要认证用户名和密码。为了方便，可以先用`adduser`创建一个账户(也可以使用已有账户)，然后把该用户添加到samba用户组中。
```shell
sudo smbpasswd -a sambauser
```
把`sambauser`添加到samba用户组中，当认证用户名和密码的时候，就可以使用sambauser的用户名和密码了。