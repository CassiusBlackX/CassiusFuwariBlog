---
title: tmux-cmds
published: 2024-09-17
description: 'frequently used tmux commands'
image: ''
tags: ["linux", "tmux"]
category: cmd
draft: false 
lang: ''
---
# tmux的作用
+ 终端复用（能够把一个终端当作好几个终端来用，影视剧中很常见的“黑客场景” `hollywood` 就是基于tmux的）
  + 可以让新窗口接入已经存在的会话
  + 允许每个会话又多个连接窗口，因此可以多人实时共享会话
  + 窗口垂直和水平分隔
  + 一个窗口同时访问多个会话

# 使用
## 会话管理
### 新建会话
```shell
tmux new-session -t ${session-name}
```
### 展示会话
```shell
tmux ls
```
展示所有tmux的会话，和相应的状态
### 分离会话
快捷键`^b d`，或者使用下面的命令
```shell
tmux detach
```
### 接入会话
```shell
tmux a -t ${session-name}
tmux a -t ${session-id}
```
其中，`session-name`或者`session-id`都可以通过`tmux ls`命令查看到

tmux同时还支持模糊识别，比如说如果你只有一个名字为"mysession"的会话，可以直接通过`tmux a -t a`接入到会话中。
### 杀死会话
```shell
tmux kill-session  # 按照id顺序杀死会话
tmux kill-session -t ${session-name}  # 杀死指定名字的会话
tmux kill-session -t ${session-id}  # 杀死指定id的会话 
```
### 切换会话
```shell
tmux switch -t ${session-id}
tmux switch -t ${session-name}
```
### 重命名会话
```shell
tmux rename-session -t ${session-id} ${session-name}
tmux rename-session -t ${old-session-name} ${new-session-name}
```

## 窗格操作
tmux允许把一个窗口分成多个窗格
### 划分窗格
+ `^b "` 将会把窗口划分为上下两个窗格
+ `^b %` 将会把窗口划分为左右两个窗格
### 移动光标
`^b` + 方位键移动光标。

## 窗口操作
tmux允许在一个会话内有多个窗口
### 新建窗口
```shell
tmux new-window -n ${window-name}
```
### 切换窗口
```shell
tmux select-window -t ${window-id}
tmux select-window -t ${window-name}
```
### 重命名窗口
```shell
tmux rename-window ${new-name}
```
### 窗口快捷键
+ `^b c`: 创建一个新窗口
+ `^b p`: 切换到上一个窗口(previous)
+ `^b n`: 切换到下一个窗口(next)
+ `^b ${number}`: 切换到指定编号的窗口
+ `^b w`: 从列表中选则窗口
+ `^b ,`: 窗口重命名