---
title: git cmds
published: 2024-04-02
description: 'basic command of git'
image: ''
tags: ["git"]
category: cmd
draft: false 
lang: ''
---

# git的相关操作
## 基础
+ `git help <command>` 获取git命令的帮助信息
+ `git init` 创建一个新的git仓库，其数据会被存放在一个名为'.git的目录下'
+ `git status` 显示当前的仓库状态
+ `git add <filename>` 添加文件到暂存区
  + `git add .` 把当前路径下的所有文件都添加到暂存你去（递归的），但是不包括'.gitignore'文件中声明的不被包含的文件
+ `git commit` 创建一个新的提交
+ `git log` 显示历史日志
+ `git log --all --graph --decorate` 可视化历史记录（有向无环图）
+ `git diff <filename>` 显示与暂存区文件的差异
+ `git diff <revisiokn> <filename>` 显示某个文件两个版本之间的差异
+ `git checkout <revision>` 更新HEAD和目前的分支
## 分支和合并
+ `git branch` 显示分支
+ `git branch <name>` 创建分支
+ `git checkout <name>` 切换到分支
  + `git checkout -b <name>` 创建分支并切换到该分支
+ `git merge <revision>` 合并分支到当前分支
+ `git mergetool` 使用工具来处理合并冲突
+ `git rebase` 将一系列补丁变基变为新的基线
## 远端操作
+ `git remote` 列出远端
+ `git remote add <name> <url>` 添加一个远端
+ `git push <remote> <local branch>:<remote branch>` 将对象传送至远端并更新远端引用
+ `git branch --set-upstream-to=<remote>/<remote branch>` 创建本地和远端分支的关联关系
+ `git fetch` 从远端获取对象/索引
+ `git pull` 相当于`git fetch; git merge`
+ `git clone` 从远端下载仓库
## 撤销
+ `git commit --amend` 编辑提交的内容或信息
+ `git reset HEAD <file>` 恢复暂存的文件
+ `git checkout -- <file>` 丢弃修改
## git高级操作
+ `git config` 可以设置git的相关参数
+ `git clone --depth=1` 浅克隆，不包括完整的版本历史信息
+ `git add -p` 交互式暂存
+ `git rebase -i` 交互式边基
+ `git blame` 查看最后修改某行的人
+ `git stash` 暂时移除工作目录下的修改内容
+ `git bisect` 通过二分查找搜索历史记录

## 子模块
子模块就是当你的项目中存在着另外一个git仓库，这种时候，如果你希望能够保持另外一个git仓库的独立性，就需要吧“另外一个git仓库”给设置为当前项目的子模块。

如果你非常希望能够删除子模块的话，直接进入到子模块文件夹中，删除`.git`文件夹就可以了。但是请务必注意是否符合相关的许可要求。

#### 克隆
在克隆一个包含子模块的项目的时候，默认的`git clone`只会克隆下来子模块所在的目录，文件夹内部并没有文件。一般来说直接通过`git clone --recurse-submodules`来要求同时克隆子模块。

#### 添加
当在你的项目内部还额外通过`git clone`克隆下来了另外一个仓库的时候，应当使用
```shell
git submodule add <submodule-url> <submodule-path>
```
来把子模块添加到本项目中。可以在`.gitsubmodules`文件中看到相关的内容。

#### 初始化和更新
为了确保子模块能够被初始化和更新，还需要运行以下两个命令
```shell
git submodule init
git submodule update
```

#### 同步
在子模块中更改完后，在主项目中直接通过`git add <submodule-path>`来进行同步

#### 更新
只是单纯的在主项目中使用`git pull`并不会把你在子模块中的更新也推送到子模块对应的仓库中。

一个普通的办法就是直接进入到子模块的目录下，手动进行`git push`，然后再回到主项目中执行推送。

当然，也可以通过以下命令让git自动尝试推送子模块，然后再推送主项目
```shell
git push --recursive-submodules=on-demand
```

## `git diff`
用来做比较的命令。在linux上系统本身还有一个`diff`命令用来比较两个不同的文件，但是`git diff`是用来进行仓库级别的比较。在windows上除了使用在线代码比较之外，还推荐使用软件`winMerge`来进行文件夹级别的比较，可以找出文件夹下每个文件的不同，在文件上也提供行级别的比较。

### 命令
```shell
git diff [<options>] [<commit>] [--] [<path>]
```
查看你相对于索引（下次提交的暂存区域）锁做的修改。（上次`git commit`和当前工作区状态的区别）
```shell
git diff [<options>] --cached [--merge-base] [<commit>] [--] [<path>]
```

```shell
git diff [<options>] [--merge-base] <commit> [<commit>] <commit> [--] [<path>]
```

```shell
git diff [<options>] <commit>...<commit> [--] [<path>]
```

```shell
git diff [<options>] --no-index [--] <path> <path>
```



## 附加小方法
在Windows上,有时会遇到无法ssh无法push上github的时候，尝试在开代理的情况下，在`~/.ssh/config`中添加以下内容
```
Host github.com
    HostName ssh.github.com
    Port 443
```
因为你的代理肯定都会照看到443端口的访问的，所以只要让ssh的时候也是走443端口就可以解决问题了。
