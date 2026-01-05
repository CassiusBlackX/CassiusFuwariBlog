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
export PATH=/root/.nvm/versions/node/v24.12.0/bin:$PATH

TARGET="/var/www/html" # ngingx/caddy所指向的网页的根目录,index.html所在的目录
GIT_DIR="/var/www/blog.git" # 用来当作git仓库的目录
WORK_TREE="/var/www/blog-deploy"

mkdir -p $WORK_TREE

git --work-tree=$WORK_TREE --git-dir=$GIT_DIR checkout -f

cd $WORK_TREE
# 保险起见,使用完整路径来使用pnpm
/root/.nvm/versions/node/v24.12.0/bin/pnpm install 
/root/.nvm/versions/node/v24.12.0/bin/pnpm build

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

# 更新博客的最新版本代码
该博客是使用的是[fuwari blog](https://github.com/saicaca/fuwari)的框架搭建的博客，目前还在积极维护中。前端时间在push到github上的时候，github actions自动检查失败。推测是本地的代码部分不符合linter或者其它的要求，所以需要更新代码部分，同时保留我自己的posts。

## 准备
为了数据安全，在进行之后的操作前，首先把本地的所有要保存的修改commit，并切换到新的分支，做接下来的修改。
```sh
git add .
git commit -m "commit before merging from fuwari upstream"
git checkout -b dev
```
## 合并上游代码
首先添加fuwari的上游github仓库的最新代码
```sh
git remote add fuwari https://github.com/saicaca/fuwari
git fetch fuwari
```
一口气都合并到当前dev分支中
```sh
git merge fuwari/main --allow-unrelated-historyies -X theirs
```
接下来就可以看到有新增文件和修改的文件。此时自己逐个选择需要恢复的文件，使用命令
```sh
git checkout HEAD~1 -- <文件路径1> <文件路径2> ...
```
就可以把要自己版本的博客内容文件恢复回来。

## 合并回自己的main分支
建议首先在本地编译并检查一下当前的效果是否正确，是否把要保留的本地文件都全部选择对了。然后仍然在dev分支中，add并提交这些修改的文件
```sh
git add <修改了的文件>
git commit -m "merge latest code from upstream"
```
然后切换回main分支并合并dev分支
```sh
git checkout main
git merge dev
git push origin main
```
等待github action自检成功并没有问题，就说明成功了。

## 可能可以使用的方法（未测试）
创建/编辑.gitattributes文件
```sh
# files that always use local version
src/content/posts/ merge=ours
src/config.ts merge=ours

# other files use default merge strategy
* merge=normal
```
定义合并驱动
```sh
git config merge.ours.driver true
```
合并
```sh
git merge fuwari/main --allow-unrelated-histories
```

# 基于github actions编译博客并部署到服务器
## 创建专用的用户并分配权限
为了安全起见，专门创建一个`deployer`用户用于在github actions中上传编译好的静态博客。
```sh
# create user
sudo useradd -m -s /bin/bash deployer
sudo mkdir -p /home/deployer/.ssh
sudo chown -R deployer:deployer /home/deployer/.ssh
sudo chmod 700 /home/deployer/.ssh
sudo chmod 600 /home/deployer/.ssh/authorized_keys
```
把`deployer`加入到`www-data`用户组
```sh
sudo usermod -aG www-data deployer
```
创建并设置blog目录的权限+setgid
```sh
sudo mkdir -p /var/www/blog
sudo chown -R www-data:www-data /var/www/blog
# setgid: new file/dir inherit from parent
sudo find /var/www/blog -type d -exec chmod 2775 {} \;
sudo find /var/www/blog -type f -exec chmod 664 {} \;
# make sure we are writable
sudo chmod -R g+rwX /var/www/blog
```
设置默认ACL，确保新文件集成租rw权限
```sh
sudo apt-get update && sudo apt-get install -y acl
sudo setfacl -R -m g:www-data:rwX /var/www/blog
sudo setfacl -R -d -m g:www-data:rwX /var/www/blog
```
## 配置github action
在`.github/workflows`中，可以新创建，也可以直接复用，新增以下job
```yaml
  deploy:
    needs: build
    runs-on: ubuntu-latest
    name: Deploy to Servers
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 24.12.0

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          run_install: false

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Build site
        run: pnpm astro build

      - name: Ensure rsync and ssh available
        run: sudo apt-get update && sudo apt-get install -y rsync openssh-client

      - name: Start ssh-agent and add key
        uses: webfactory/ssh-agent@v0.5.3
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Add remote hosts to known_hosts
        env:
          REMOTE_HOSTS: ${{ secrets.REMOTE_HOSTS }}
          SSH_PORT: ${{ secrets.SSH_PORT }}
        run: |
          for host in $(echo "$REMOTE_HOSTS" | tr ',' ' '); do
            port="${SSH_PORT:-22}"
            ssh-keyscan -p "$port" -H "$host" >> ~/.ssh/known_hosts || true
          done

      - name: Rsync dist/ to each remote host
        env:
          REMOTE_HOSTS: ${{ secrets.REMOTE_HOSTS }}
          REMOTE_USER: ${{ secrets.REMOTE_USER }}
          REMOTE_PATH: ${{ secrets.REMOTE_PATH }}
          SSH_PORT: ${{ secrets.SSH_PORT }}
        run: |
          set -e
          for host in $(echo "$REMOTE_HOSTS" | tr ',' ' '); do
            port="${SSH_PORT:-22}"
            ssh -p "$port" -o StrictHostKeyChecking=yes "${REMOTE_USER}@${host}" "mkdir -p '${REMOTE_PATH}'"
            rsync -avz --delete --omit-dir-times --no-perms --no-owner --no-group  --chmod=Du=rwx,Dg=rwx,Fu=rw,Fg=rw -e "ssh -p $port" dist/ "${REMOTE_USER}@${host}":"${REMOTE_PATH}"
          done

```
需要在`settings`中设置的环境变量有
+ `REMOTE_HOSTS`:逗号分隔的VPS的IP地址
+ `REMOTE_PATH`: 部署博客的位置（如果按照上面的命令，那么就是`/var/www/blog`目录）
+ `REMOTE_USER`: 用于ssh的用户（按照上面的命令就是`deployer`）
+ `SSH_PORT`: 一般就是22，除非有特殊情况
+ `SSH_PRIVATE_KEY`: ssh私钥，cat出来的所有内容直接复制粘贴进去

然后commit并push，就会自动运行并上传到服务器了。如果遇到了问题，github会邮件通知。