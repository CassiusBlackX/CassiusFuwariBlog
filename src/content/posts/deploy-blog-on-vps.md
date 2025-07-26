---
title: deploy blog on vps servers
published: 2025-07-26
description: '在私有vps上部署个人博客'
image: ''
tags: ["web", "git"]
category: deployment
draft: false 
lang: ''
---
# 在公网vps服务器上部署自己的博客
## 前言
反向代理、https证书申请等问题将不在本博客中介绍.

我目前使用caddy作为反向代理服务器,Caddyfile的配置非常容易,而且能够自动申请证书,所以也不要去搞nginx+acme.sh之类的复杂的东西了

我使用的博客是astro模版的,因此以astro模版为例

## 环境准备
在服务器上要准备一个node.js和pnpm的环境. 本机能够通过公钥认证的方式能够连接上vps服务器.

## 服务器上的准备
### 创建git仓库
在服务器上选一个位置(为了符合FHS规范,最好就放在`/var/www/`下)下,创建一个git仓库`blog.git`
```shell
cd /var/www/blog.git
git init --bare
git config --bool core.bare true
git branch -m main. # 因为我在本地使用的就是main作为默认分支
# OPTIONAL: git config --global init.defaultBranch main
touch hooks/post-receive
chmod +x hooks/post-receive
```
### 配置git-hook自动化脚本
接下来使用自己熟悉的文本编辑器编辑`hooks/post-receive`文件
```bash
#!/bin/bash

# 把pnpm和node所在的路径添加到环境变量中
export PATH=/root/.nvm/versions/node/v22.17.1/bin:$PATH

TARGET="/var/www/html" # ngingx/caddy所指向的网页的根目录,index.html所在的目录
GIT_DIR="/var/www/blog.git" # 用来当作git仓库的目录
WORK_TREE="/var/www/blog-deploy"

mkdir -p $WORK_TREE

git --work-tree=$WORK_TREE --git-dir=$GIT_DIR checkout -f

cd $WORK_TREE
# 保险期间,使用完整路径来使用pnpm
/root/.nvm/versions/node/v22.17.1/bin/pnpm install 
/root/.nvm/versions/node/v22.17.1/bin/pnpm build

mkdir -p $TARGET
# 使用rsync来复制文件,减少不必要的重复覆盖,也能够自动删除不再存在的文件
rsync -av --delete $WORK_TREE/dist/ $TARGET/ || exit 1
```
其中尤其重要的是,hooks中的脚本,它的默认执行环境和当前用户(不管是root还是普通用户)是不一致的,因此如果使用的是nodejs官网上提供的安装方式,把node和pnpm安装到的变量是`/root/.nvm`中或者其它位置,不是对所有用户都一致可见的路径中,在执行post-receive的时候是没有办法执行的pnpm命令的!

## 本机上的准备
### 添加远程仓库
把vps上的git仓库作为一个远程仓库添加到本地仓库中
```shell
git remote add blog root@your.vps.address:/var/www/blog.git
```
在这里远程仓库使用的名称为blog,这样的话,正常的推送到github的时候就可以继续使用origin
### 推送到vps服务器上
```shell
git push blog main
```
把本地的main分支推送到远程,可以在输出中看到,当文件上传完以后,post-receive脚本被自动执行.并且对应的脚本执行时候的输出会在本地展示,输出的每行前面会有remote:标识.

没有遇到报错就说明成功了