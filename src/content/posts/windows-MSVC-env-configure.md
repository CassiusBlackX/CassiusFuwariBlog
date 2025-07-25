---
title: configure MSVC env on windows
published: 2025-07-25
description: '在Windows电脑上配置MSVC环境变量'
image: ''
tags: ["windows", "C"]
category: 'deployment'
draft: false 
lang: ''
---

# Windows 配置MSVC环境变量
## 安装Visual Studio
直接使用Visual Studio Installer安装就可以，安装的时候注意一定至少要勾选一个`Windows SDK`，否则会连`stdlib.h`之类的最基本的头文件都没有的。

同时也建议可以直接安装vcpkg（C++包管理器）和cmake（跨平台的C/C++项目构建工具），并配置对应的环境变量。

## cl的配置
cl就是MSVC中的编译器，等价于gcc。

cl的路径在`%MSVC_INSTALL_PATH%\VC\Tools\MSVC\${VERSION}\bin\Hostx64\x64`中。可以直接使用Everything搜索`cl.exe`，然后找到看着像的目录就可以了。（最好使用Hostx64/x64下的cl.exe会比较好）。

## INCLUDE的配置
在环境变量中新建项`INCLUDE`，并对应添加以下几个内容
+ `%MSVC_INSTALL_PATH%\VC\Tools\MSVC\${VERSION}\include`
+ `%WINDOWS_KITS_PATH%\10\Include\10.0.19041.0\cppwinrt`
+ `%WINDOWS_KITS_PATH%\10\Include\10.0.19041.0\\shared`
+ `%WINDOWS_KITS_PATH%\10\Include\10.0.19041.0\ucrt`
+ `%WINDOWS_KITS_PATH%\10\Include\10.0.19041.0\um`
+ `%WINDOWS_KITS_PATH%\10\Include\10.0.19041.0\winrt`

如果是一行的那种样式，则每个环境变量之间使用英文分号隔开

## LIB的配置
在环境变量中新建项`LIB`，并对应添加以下几个内容
+ `%MSVC_INSTALL_PATH%\VC\Tools\MSVC\14.44.35207\lib\x64`
+ `%WINDOWS_KITS_PATH%\10\Lib\10.0.19041.0\ucrt\x64`
+ `%WINDOWS_KITS_PATH%\10\Lib\10.0.19041.0\ucrt_enclave\x64`
+ `%WINDOWS_KITS_PATH%\10\Lib\10.0.19041.0\um\x64`

注意这里的架构要和前面选择的cl的架构统一。如cl选择的是x64的，那么这里也就要对应选择x64的

## cmake的配置
cmake会位于`%WINDOWS_KITS_PATH%\Common7\IDE\CommonExtensions\Microsoft\CMake\CMake\bin`中，只要把这个变量添加到环境变量中即可

## vcpkg的配置
vcpkg会位于`%WINDOWS_KITS_PATH%\VC\vcpkg`中