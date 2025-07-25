---
title: jetson ros opencv cuda compatible
published: 2024-06-09
description: '在jetson开发板上源码编译opencv，让opencv能够是用cuda加速'
image: ''
tags: ["ros", "opencv", "jetson"]
category: deployment
draft: false 
lang: ''
---

## ros换源
原中科大源很多包都已经不再被维护了，使用清华源
```bash
sudo sh -c '. /etc/lsb-release && echo "deb http://mirrors.tuna.tsinghua.edu.cn/ros/ubuntu/ $DISTRIB_CODENAME main" > /etc/apt/sources.list.d/ros-latest.list'
sudo apt update
```

## opencv
因为jetson xavier nx自带的opencv没有支持cuda，为了能够使用cuda，需要源码编译opencv4.5。

但是在我的情况下，我的ros的工作空间中有太多的文件已经依赖自带的opencv了，所以不能直接删除自带的opencv而完全替换为新的opencv4.5.4。因此需要能够同时允许系统自带opencv和支持cuda的opencv4.5.4

### opencv4.5.4编译
从opencv官网下载opencv4.5.4的源码，和对应的opencv_contribe的源码，解压。
为了不影响本来就有的opencv，在`/usr/local/`下自己新建一个`opencv4.5.4`的文件夹
```bash
sudo mkdir /usr/local/opencv4.5.4
```
进入到解压后的`opencv-4.5.4`文件夹中，自己新建一个`build`文件夹，进入其中，使用以下命令。
```bash
cmake \
-DCMAKE_BUILD_TYPE=Release \
-DCMAKE_INSTALL_PREFIX=/usr/local/opencv4.5.4 \
-DOPENCV_ENABLE_NONFREE=1 \
-DBUILD_opencv_python2=1 \
-DBUILD_opencv_python3=1 \
-DWITH_FFMPEG=1 \
-DCUDA_TOOLKIT_ROOT_DIR=/usr/local/cuda \
-DCUDA_ARCH_BIN=7.2 \
-DCUDA_ARCH_PTX=7.2 \
-DWITH_CUDA=1 \
-DENABLE_FAST_MATH=1 \
-DCUDA_FAST_MATH=1 \
-DWITH_CUBLAS=1 \
-DOPENCV_GENERATE_PKGCONFIG=1 \
-DOPENCV_EXTRA_MODULES_PATH=../../opencv_contrib-4.5.4/modules \
..

```
其中，`CMAKE_INSTALL_PREFIX`就是希望把opencv按转到的目标路径，`OPENCV_EXTRA_MODULES_PATH`是对应的`opencv_contrib`中的module文件夹的相对路径。

在`cmake`成功后，就可以`make`了。

因为是在jetson板子上进行编译，所以需要的时间可能会非常长，又不想费精力去弄交叉编译了，所以要尽可能的利用多核，在我这里我使用`make -j6`因为nx有6个CPU核心，同时开了一个`tmux`窗口，这样可以防止不小心把make给挂了。

在`make`结束后还需要`sudo make install`，把一些相关的库放到`/usr`等系统目录下。

### 对应`cv_bridge`的处理
melodic自带的cv_bridge和opencv4.5会有冲突，所以如果希望使用opencv4.5，需要自己在工作空间下下载源码vision_opencv，然后使用工作空间内的cv_bridge。

为了能够使用opencv4.5，需要下载[noetic 版本的vision_opencv](https://github.com/ros-perception/vision_opencv/tree/noetic),并且修改`vision_opencv/cv_bridge`下的CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.0.2)
project(cv_bridge)

set(OpenCV_DIR "/usr/loca/opencv4.5.4")  # 此处OpenCV_DIR修改为自己安装的opencv的路径

find_package(catkin REQUIRED COMPONENTS rosconsole sensor_msgs)

if(NOT ANDROID)
  find_package(PythonLibs)

  if(PYTHONLIBS_VERSION_STRING VERSION_LESS "3.8")
    # Debian Buster
    find_package(Boost REQUIRED python3)
  else()
    # Ubuntu Focal
    find_package(Boost REQUIRED python)
  endif()
else()
find_package(Boost REQUIRED)
endif()

set(_opencv_version 4)
find_package(OpenCV 4.5.4 QUIET)  # 为了明确指定希望使用的opencv的版本
if(NOT OpenCV_FOUND)
  message(STATUS "Did not find OpenCV 4, trying OpenCV 3")
  set(_opencv_version 3)
endif()
```
但是仅仅修改以后，cmake虽然可以了，但是在编译的时候仍然会遇到报错“ 
/home/nvidia/zal_ws/src/vision_opencv/cv_bridge/src/module.hpp: In function ‘void* do_numpy_import()’:
/usr/include/python2.7/numpy/__multiarray_api.h:1537:144: error: return-statement with no value, in function returning ‘void*’ [-fpermissive]
 #define import_array() {if (_import_array() < 0) {PyErr_Print(); PyErr_SetString(PyExc_ImportError, "numpy.core.multiarray failed to import"); return NUMPY_IMPORT_ARRAY_RETVAL; } }
”

根据报错信息，直接去修改源码：

进入`/vision_opencv/cv_bridge/src/module.hpp`，把`do_numpy_import`函数的声明修改为
```c++
static void do_numpy_import( )
{
    import_array( );
}
```
再次`catkin_make`，成功编译。

### 在工作空间中指定需要的cv_bridge和opencv
在自己的`CMakeLists.txt`中明确指定需要的opencv版本,cv_bridge不需要专门指定，`catkin_make`的时候会优先使用工作空间内的`cv_bridge`的源码编译生成的包的。

```cmake
set(OpenCV_DIR "/usr/local/opencv4.5.4")  # Opencv的安装路径
find_package(OpenCV 4.5.4 REQUIRED)

find_package(catkin REQUIRED COMPONENTS
  cv_bridge
  ... # 其它需要的包
)
```
接下来，在代码中，就可以正常的导入`opencv4.5`和`cv_bridge`了。