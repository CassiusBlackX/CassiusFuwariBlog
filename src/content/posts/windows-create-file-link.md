---
title: windows-create-file-link
published: 2025-10-07
description: Windows上创建符号链接、软链接、影链接
image: ''
tags: ['Windows']
category: cmd
draft: false 
lang: ''
---
# Windows创建链接
## 命令
```cmd
mklink [/d][/h][/j] 目标路径 源路径
```
+ 参数解析
  + `/d`: 创建目录符号链接（软链接的时候应当始终优先使用这个）
  + `/h`: 创建硬链接（必须在同一个硬盘分区上，只能是文件，不能是目录）
  + `/j`: 创建目录链接（老旧方式，不需要使用）