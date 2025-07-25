---
title: vcrypto-paper-reading
published: 2025-07-25
description: ''
image: ''
tags: ["linux", "virtualization"]
category: notes
draft: false 
lang: ''
---
# introduction
+ vcrypto: 一个半虚拟化框架，适配各种加解密资源，用来提升云服务中的网络安全性能
+ 克服的挑战
  + 在卸载任务到加速器之前，确定哪些加速器能够执行哪些算法
    + session cache + reuse机制
    + 事件驱动的消息提示机制，直接通过虚拟地址可以定位卸载了的请求
    + 重新设计了openssl engine,分离为了前后端
  + 把多种加解密资源分配为组，并调度这些资源，充分利用
  + 用两层的轮询机制+内存共享机制来接近裸机的性能

# background
## Secure sockets layer
+ TLS连接的过程
  + 握手阶段，客户端和服务端协商是用的加密算法和Message authentication code算法，信任证书并生成四份密钥
  + 数据传输阶段，是用私钥对称加密并传输用户数据
    + 传输时候的数据大小不会超过16KB，超过的会被切分到16KB一块
+ 结论
  + 当数据量较小，瓶颈主要集中在握手阶段的非对称加解密
  + 当数据量较大，瓶颈集中在数据传输阶段

## OpenSSL framework
### 软件栈
+ Application
+ SSL/TLS API
+ libssl: 共享库，提供SSL/TLS API给用户程序
+ ↓
+ EVP API: 封装libcrypto 
+ libcrypto: 提供各种加解密算法的实现
+ Engine APi
+ Engine DLL: 注册到libcrypto中，把能够卸载到硬件加速的任务卸载

### asynchronous request offloading
+ fiber: 比线程更轻量级，上下文切换开销更小，提供异步IO
+ 过程
  + 应用程序调用OpenSSL
  + OpenSSL通过`ASYNC_start_job`函数来开启多个fiber，并为每个fiber分配一个file descriptor(fd)
  + fiber异步的把负载卸载并调用`ASYNC_pause_job`函数来把控制权返回给应用程序进程
  + 应用程序进程通过IO多路复用的方式监听每个fd
  + 当一个请求完成，engine层或者驱动会给对应的fd写一个事件，来通知应用程序进程
  + 应用程序会再次唤醒`ASYNC_start_job`来恢复到fiber的上下文中并提取执行结果
  + fiber完成后自动推出，释放资源

## crypto device cirtualization
+ 现有的应用程序都是调用OpenSSL API，难以从加速器，或者其配套的DLL中受益
+ 直接device-passthrough的话受限于云服务器厂商所能够提供的加速设备和驱动，可选择少，且配置麻烦
+ device-passthrough会让系统对于硬件任务失败非常敏感，降低系统健壮性

## targets and challenges
### stateful crypto requests
加密设备的虚拟化必须有状态
+ 基于metadata的请求
  + 应用程序的一个请求是包含metadata和userdata两部分
  + 在卸载请求到加速器设备之前，必须要检查metadata中的内容，判断是否能够卸载到加速器
  + 对于大数据请求，必须要把超过16KB的用户请求拆分成多个相同metadata的请求
+ 请求结果回收
  + 在不依赖外部卸载且严格保证响应顺序一致性的前提下，通知请求发起方获取结果
+ 当动态库被多个应用程序加载的时候
  + 要避免锁竞争
  + 维持高吞吐下的线程扩展能力

### robust crypto service
+ 能够支持不同加解密算法的加速器设备之间要能够协同工作，以应对来自用户的各种类型的请求
+ 多个、多种加速器的调度

### efficient IO virtualization
+ VM-EXITS带来的上下文切换开销和缓存污染
  + 主要来自PV的前后端之间的通信
+ 内存拷贝开销
  + PV前后端之间的数据传输依赖内存拷贝，不能直接访问

# design
## overview
### vcrypto-engine
+ engine本质上是一个OpenSSL engine的魔改。
+ engine收集来自各应用程序的请求，并转发到virtio cryptodev driver
+ 当请求完成后，通知应用程序
+ 为了在卸载负载的时候不要阻塞其他请求，基于OpenSSL的fiber, engine支持异步卸载
### virtio cryptodev driver
+ 用户空间的virtio 轮询模式的驱动
+ 为客户应用程序提供API
+ 把session request或者crypto request卸载下去
  + session request: 只有metadata
  + crypto request: 同时包括metadata和userdata
### hypervisor
+ 拦截来自客户机虚拟机的virtio控制消息，并转发到vcrypto-backend。
+ 当hypervisor收到的是session request的时候，hypervisor会像vcrypto-backend发送一条消息让它创建一个session opbject，并把对应的sessino ID 返回给PV的前端
### vcrypto-backend
+ 与virtio cryptodev driver通过vhost-user协议交互，用于
  + 虚拟设备的生命周期管理: 初始化 & 销毁
  + 请求传输: session requst & crypto request
+ 队列架构：
  + control queue: 传输sessino request,
  + data queue

## session management
把crypto metadat抽象为session object
### creation of sessino objects
+ vcrypto-engine唤醒virtrio cryptodev driver，创建一个session request(session request包含crypto metadata和一个空的session id)
+ 添加session request到control queue中
+ 通过VM-EXITS通知hypervisor
+ hypervisor解析session request，提取出crypto metadata并用unix socket发送给vcrypto-backend
+ vcrypto-backend在本地创建一个新的session object，并把这个Object添加到本地的哈希表中并把object的session id返回给hypervisor
+ hypervisor把session id填充到session request中并把这个request的状态标记为已完成
+ session id被驱动返回到vcrypto-engine中
+ 如果之后的请求也是相同的metadata的话，那么他们的session id也将会保持相同
### cache for sessino objects
+ 是一个hashmap，位于engine前后端之间的共享内存中
+ key: crypto metadata的MD5结果
+ value: 对应的session id
### destruction of session objects
+ 对每个session object采用引用计数，由engine backend负责
+ 当engine frontend发出了一个session id的请求，引用计数加一
+ 当请求完成，引用计数减一
+ 引用计数为0，cache中的记录删除，session object也析构

## crypto requests offloading and respoinse retrieve
### engine layer
+ engine前端连接到每一个需要的应用程序进程
+ 添加一个transport queue来在engine前后端之间传输crypto request
+ 客户机的进程可以无需临界区保护地向自己的trans queue添加请求。
+ trans queue是一个无锁的唤醒缓冲区，在前后端的共享内存中

### engine frontend
+ 一个动态库，类似OpenSSL engines
+ 当被客户进程加载的时候，船舰一条unix socket通信到engine后端
+ engine后端创建一个trans queue并返回对应的配置(start virtual addr & queue length)
+ 客户程序是用标准OpenSSL API就可以把crypto request卸载到engine前端上
+ 一个fiber一次会卸载一个crypto request,并且记录这个crypto request object的地址。（其中crypto request obeject是由前端创建的）
+ 多个fiber可以异步的卸载crypto requests
+ 为了能够向下卸载cyrpto request,前端首先会创建crypto request object。
+ 前端根据crypto request中的metadata计算MD5值，并以此作为索引询问engine backend是否存有对应缓存
  + 缓存命中，把这个session id添加到crypto request object中对应字段，并且把crypto request添加trans queue中
  + 缓存不命中，engine backend会创建一个session request。
+ 当应用程序发布玩所有的request之后会sleep
+ 前端为每一个crypto request添加一个fiber fd，当请求完成，会通过此来唤醒应用程序并且恢复fiber上下文
+ 请求的完成结果可以由fiber从crypto request object中是用虚拟地址就可以读取到，不需要再卸载的操作，并且能够严格保持响应顺序的一致性  // BUG: 怎么就保证了相应顺序的一致性了？

### engine backend
+ 一个守护进程，连接engine前端和virtio cryptodev driver
+ 任务
  + 当发生了session cache miss,调用virtio cryptodev driver API，创建一个session request,并添加到hypervisor的control queue中
  +  收集trans queue中的所有crypto request，调用virtio cryptodev driver batch processing API来分批次的把请求通过data queue发送给vcrypto backend
  +  调用driver API来轮询每个data queue中每个request的状态，并通过fiber fd来通知应用程序进程去取请求的执行结果

### vcrypto backend
+ 从data queue中提取crypto requests
+ 在本地创建crypto request并发送到调度器层，调度器把请求发送到合适的加速器上
  + 创建一个本地crypto request:
    + 从data queue中获取“纯粹的”crypto request
    + 从session hashmap中提取session object,其中，使用session id作为key
+ 以batch为单位，从加速器驱动中提取crypto reques的执行结果并更新requests的状态为完成或失败。

## heterogeneous cryptographic resource pool
### flexible scheduler layer
+ 在各种加速资源中分发crypto request
+ 三种基础策略
  + capability oriented: 哪个资源能够加速什么算法，就分配给它
  + round-robin: 当多个资源都能够加速一个算法的时候，轮转调度
  + master-worker: 再把资源进一步细分为primary group和secondary group
    + 优先调用primary group中的资源
    + 当primary group中的资源都忙，再调用secondary
    + primary有空了，又优先回去调用primary的资源

## two layer polling
+ 在engine backend中的轮询线程的任务
  + 顺序收集来自各个应用程序的请求
  + 调用virtio cryptodev driver API并把requests添加到data queue中
  + 调用driver API把完成了的请求结果提取出来
+ vcrypto backend中的轮询线程的任务
  + 收集来自data queue中的请求
  + 创建本地请求并把请求卸载到加速器设备上
  + 从加速器设备上获取请求的执行结果，收集并更新每个请求的状态
+ 以上两个轮询都只在高负载状态下才启用，否则采用的是doorbell notification mechanism
+ // TODO 如何侦测是否是高负载的？

# implementation
## crypto requests format and resource pool
+ `rte_crypto_op`: 代表一个crypto request
  + `mempool`: 代表这个结构体所分配在的内存池是哪一块
  + `rte_crypto_sym_op`:
    + `m_src`: 要处理的数据片段的buffer地址
    + `m_dst`: 复用`m_src`的buffer，来保存处理结果
    + `rte_cryptodev_sym_session`: crypto metadata

### opdone structure
是用opdone结构体，把fiber fd集成到每个crypto request中，以实现完成通知。

opdone结构体同时包含已经提交的请求书和已经处理的请求数，以支持openssl的流水线模式。  
在这个模式中，fiber只有在持续提交请求到了16个以后才会挂起。并且要所有的请求都被处理完成后，fiber才会被唤醒。  
应用程序激活流水线模式的时候，会同时创建多个crypto request structrue, 并共享同一个opdone结构体

### crypto request structure memory pool
+ 为了减少频繁的内存分配，也防止内存碎片化。engine backend维护了多个预分配的内存池，来存储crypto requests structures。
+ `mempool`的结构
  + `Initialization Vectors`: 专用CBC模式对称加密算法的初始化向量存储
  + 填充字段: 把结构体大小填充到和cacheline一样大
+ 在引擎初始化的时候可以调整memory pool的数量和每个memory pool的请求大小限制。

## virtio abstract of crypto request