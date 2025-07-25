---
title: configure MSVC env on windows
published: 2025-07-25
description: '在Windows电脑上配置MSVC环境变量'
image: ''
tags: ["windows", "cpp"]
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
vcpkg会位于`%WINDOWS_KITS_PATH%\VC\vcpkg`中，建议直接新建一个变量`VCPKG_ROOT`，把这个变量的值设置为vcpkg的路径

### cmake+vcpkg的使用
在使用cmake构建项目，并且依赖vcpkg下载第三方库的时候,CMakeLists.txt要做如下的配置。其中`$ENV{VCPKG_ROOT}`能够直接捕获环境变量中`VCPKG_ROOT`的值，这样的话只要系统的环境变量配置正确，就一定可以确保vcpkg是可用的了。
```cmake
cmake_minimum_required(VERSION 3.10)

if(DEFINED ENV{VCPKG_ROOT})
    set(CMAKE_TOOLCHAIN_FILE "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
else()
    message(STATUS "VCPKG_ROOT is not defined, using system libraries if available.")
endif()

# `CMAKE_TOOLCHAIN_FILE` must be configured before `project`, or it is not going to work
project(your_project_name)
```

# 解决在vscode中，使用cmake构建项目，使用MSVC编译时候，中文输出乱码的问题
进入vscode设置，直接查找`Cmake: Output Log Encoding`，改为utf-8即可