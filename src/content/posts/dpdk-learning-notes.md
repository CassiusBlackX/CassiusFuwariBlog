---
title: dpdk-learning-notes
published: 2025-08-02
description: ''
image: ''
tags: ["dpdk"]
category: 'notes'
draft: true 
lang: ''
---
# overview
+ 主要目标: 提供一个框架,能够快速高效处理数据包
+ DPDK框架创建了一系列的库,针对特定的环境,但是都抽象到EAL中
+ DPDK绝大部分的程序都使用轮询来提升性能,但是部分会使用中断或者事件驱动

## Environment Abstraction Layer (EAL)
提供一个通用的接口,隐藏具体环境相关的需求,提供给普通的用户应用程序和库

## core components
### ring manager
+ 提供一个无锁的FIFO,多生产者、多消费者的有限大小的表.
+ `Memory Pool Library`中使用的就是ring
### memory pool manager
+ 一个pool是由name来标识的,用ring来存储free objects
### network packet buffer management
+ 提供用来给DPDK存储消息buffer的创建和销毁buffer的功能
### time manager

## ethernet pool mode driver architecture

# memory management
## Lcore Variables
### lcore variables API
每一个lcore variable都针对每一个EAL线程和注册了的非EAL线程都有一个独特的值.因此对于每一个过去、现在和之后的lcore id-equipped线程,都会有一个独特的值. 上限为`RTE_MAX_LCORE`

#### lcore variable handle
使用一个`handle`来分配和管理一个每一个lcore variable的值. `handle`本身是一个opaque指针,只用使用`rte_lcore_var.h`中的宏才能够正确的解引用

`handle`的底层数据结构是一个指向对应value的指针(如如果lcore variable本身是一个`uint32_t`的变量,那么`handle`就会是一个`uint32_t*`).这样能够避免使用void指针,有效利用类型检查机制

一个有效的`handle`永远不会是NULL. 所以NULL可以用来代表这个handle的分配还没有完成

#### lcore variable allocation
+ 创建步骤
  + 定义一个lcore variable handle,使用`RTE_LCORE_VAR_HANDLE`
  + 分配存储空间,并初始化`handle`,使用`RTE_LCORE_VAR_ALLOC`或者`RTE_LCORE_VAR_INIT`.
+ lcore variable的生命周期并不和它对应的那个线程绑定
+ 每个lcore variable都有`RTE_MAX_LCORE`个值,每一个值对应一个可能的lcore id.
+ 只要lcore variable被创建了,在整个生命周期中都可以被读取,直到使用诸如`rte_eal_cleanup()`之类的函数结束
+ lcore variable不需要也不能被手动释放

#### access
+ 虽然lcore variable可以在任意时候被任何一个线程读,但是它应当仅被它对应的那个线程频繁的读写
+ 无主访问会导致*不正确的共享*,但是这个情况比较罕见,所以对程序行为不会有什么影响
+ 对于同一个lcore variable,对应的不同的lcore id的值,可能会被对应的线程频繁的读写,但是这样不会导致*不正确的共享*
+ 在主人线程和非主人线程访问同一个lcore variable的时候,应当使用正确的同步机制,防止产生数据竞争
+ 对一个lcore variable的一个特定的lcore id的访问通过`RTE_LCORE_VAR_LCORE`
+ 如果是主人线程访问自己的lcore variable value的话,可以通过`RTE_LCORE_VAR`来访问
+ `RTE_LCORE_VAR_FOREACH`能够用来便利一个lcore variable的所有值
+ `handle`由`RTE_LCORE_VAR_HANDLE`定义,可以被当作对应值的指针,但是必须使用前文提到的*正确的方式*解引用

#### storage
+ lcore variable的值可能是基础数据类型如`int`,但本质上来说是一个结构体`struct`
+ lcore variable的大小不可以超过DPDK设置的最大值`RTE_MAX_LCORE_VAR`

### implementation
// TODO: 没有记录