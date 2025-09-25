---
title: vcrypto-paper-reading
published: 2025-07-25
description: '对论文vCrypto: a Unified Para-Virtualization Framework for Heterogeneous Cryptographic Resources的阅读笔记'
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
### cache for session objects
+ 是一个hashmap，位于engine前后端之间的共享内存中
+ key: crypto metadata的MD5结果
+ value: 对应的session id
### destruction of session objects
+ 对每个session object采用引用计数，由engine backend负责
+ 当engine frontend发出了一个session id的请求，引用计数加一
+ 当请求完成，引用计数减一
+ 引用计数为0，cache中的记录删除，session object也析构

## crypto requests offloading and response retrieve
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


# UVCrypto
## 工作
### 目标和内容
+ 通过采用半虚拟化框架，消除基于物理硬件的虚拟化支持，使得虚拟设备和物理设备能够完全解耦，从而使得虚拟设备的数量不受限制
+ 通过将半虚拟化前端集成到加解密算法库中，使得客户机应用程序能够方便地从使用异构加解密资源中收益，而无需修改代码或添加新的抽象层
+ 通过在半虚拟化后端为不同的异构加解密资源构建资源池，并调度多个物理硬件设备共同协作支持同一个虚拟设备，从而扩展虚拟设备的功能、性能和健壮性，更能够满足云计算环境中客户机应用程序的复杂多样的需求
+ 能够接近SRIOV的性能
### 待解决的问题
+ 加解密技术的执行过程涉及到加解密会话状态的变化，然而传统的半虚拟化前后端通信是没有状态的，如何在半虚拟化前后端维护加解密会话状态的一致性
  + 大块数据要拆分后才能够使用硬件加速加解密，但是返回上层的时候要统一成一份数据，因此需要维护状态
+ 控制碰面上的通知机制和终端机制会触发VM-Exit，频繁的VM-Exit会导致半虚拟化框架的性能受损，如何在重负载下尽可能减少VM-Exit次数
+ 多个物理硬件设备共同支持一个虚拟设备具有更高的性能，但协同工作可能导致加解密请求的完成顺序不同于下发顺序，如何正确处理加解密请求的乱序执行现象避免被客户机应用程序观察到
+ 多个应用进程同时工作时，加解密请求的下发可能会相互之间形成竞争，如何批量处理加解密请求的下发从而减轻半虚拟化前端的压力
+ 利用一部模式处理加解密请求的卸载具有更高的性能，但需要轮询线程监测请求完成状态，如何避免在每个应用进程上都创建轮询线程带来的高昂CPU开销
+ 提供于主流开源加解密算法库相同的接口给客户机应用程序，能够使其方便的从异构加解密资源中受益，如何在半虚拟化前端像主流开源加解密算法库一样构造和模拟加解密请求
### 解决思路
+ 通过在半虚拟化前后端分别根据加解密元数据创建本地会话对象，在半虚拟化后端维护会话表单，同时以相同且唯一的会话ID作为标识为半虚拟化前后端的会话对象建立索引，使得在一次加解密请求的完整执行过程中都能够通过会话对象获取到相同的加解密元数据，在半虚拟化前后端维护了加解密会话状态的一致性
+ 通过在半虚拟化前端使用轮询模式代替传统终端模式来监测加解密请求的完成状态，消除在控制平面上的中断机制导致的VM-Exit；通过在半虚拟化后端制动轮询virtqueue获取下发的加解密请求，消除了控制平面上的通知机制导致的VM-Exit；中二实现了在重负载下尽可能的减少VM-Exit次数
+ 通过在半虚拟化前端根据地址代替顺序来标识加解密请求，避免客户机应用程序强依赖于加解密请求的顺序来获取加解密结果；通过在半虚拟化后端的资源池调度驱动层中添加顺序排序层，基于地址提供FIFO的加解密服务；从而正确处理加解密请求的乱序现象避免被客户机应用程序观察到
+ 通过在半虚拟化前端的引擎前端和引擎后端之间维护数据传输队列trans，使得引擎后端能够累计一定数量的加解密请求批处理下发，使得半虚拟化前端仅有一个virtqueue的生产者，从而减轻半虚拟化前端的压力
+ 通过将半虚拟化的前端涉及为前后端架构，将OpenSSL引擎解耦成由每个用应用进程加载的多个引擎前端和获取加解密请求并下发的单个引擎后端守护进程，将轮询的CPU开销显示在单个引擎后端守护进程上，从而避免在每个应用进程上都创建轮询线程带来的高昂的CPU开销
+ 通过将半虚拟化前端的引擎前端实现为一个动态链接库并注册到OpenSSL引擎中，使得呈现给用户进程的形式上于常见的OpenSSL引擎保持一致，从而能够像主流开源加解密算法库一样构造和模拟加解密请求

## 挑战
### 维护会话状态的一致性
加解密技术的执行过程涉及到加解密会话状态的变化，然而传统的半虚拟化前后端通信是没有状态的。

一般的，应用程序在执行加解密技术时会分为两个阶段，在第一阶段先通过包括加解密算法、加解密密钥大小等在内的加解密元数据构建加解密会话，在第二个阶段加解密会话中提供的加解密元数据依次对一连串待加解密数据块进行加密，与此同时，加解密会话还会记录加解密执行是否成功、加解密执行的成功次数等状态信息。基于virtio协议和virtqueue的半虚拟化是最典型的半虚拟化方案，每一个virtqueue仅需通过描述符列表维护可用环形队列和已用环形队列表即可，不涉及状态的维护。

### 减少VM-Exit的次数
控制平面上的通知机制和终端机制会触发VM-Exit，繁重的VM-Exit会导致半虚拟化框架性能受损，如何在中负载下尽可能减少VM-Exit次数，尽管半虚拟化框架相对于纯软件形式模拟的I/O全虚拟化技术已经减少了虚拟机和Hypervisor之间的VM-Exit次数，，但相对于以SR-IOV为代表的I/O直通技术仍然有一定的差距。

在半虚拟化框架中，VM-Exit主要存在于控制平面上的通知机制和中断机制；在通知机制产生效应的期间，EM-Exit主要由半虚拟化前端触发，用于告知半虚拟化后端它已经将请求添加到对应的virtqueue中；在终端机制产生效应的期间，VM-Exit主要由半虚拟化后端触发，用于告知半虚拟化前端它已经完成请求的卸载处理；每次产生VM-Exit都会使得程序吹欻看从客户机陷入到Hypervisor中，从而以缓存污染和性能开销为代价实现资源管理。

### 避免请求处理乱序
多个物理硬件设备共同支持一个虚拟设备具有更高的性能，但协同工作可能导致加解密请求的完成顺序不同于下发顺序，如何正确处理加解密请求的乱序执行现象避免被客户机应用程序观察到？

### 批处理下发请求
多个应用进程同时工作时，加解密请求的下发可能会相互之间形成竞争，如何批量处理加解密请求的下发从而减轻半虚拟化前端的压力？

+ 如果加载引擎的每个应用进程都调用半虚拟化前端驱动提供的接口执行卸载，会存在两个问题
  + 加解密请求的下发本质上是串行的，每个应用进程都会作为virtqueue的生产者，从而造成频繁的竞争
  + 异构加解密设备的利用率会偏低，virtqueue执行添加新元素的时间是相对分散的，难以充分利用异构加解密资源提供的批处理优化
+ 导致加解密请求下发的性能瓶颈会落在半虚拟化前端驱动向virtqueue添加元素上

### 降低轮询线程开销
异步模式处理加解密请求的卸载具有更高的性能，但需要轮询线程监测请求的完成状态，但是轮询会带来高昂的cpu开销。

同步处理模式的住线程在下发请求后会等待直到当前正在处理的请求返回，异步处理模式的线程在下发请求后会保存上下文并直接返回，由子线程继续处理当前请求，因此住线程需要一种机制来监测子线程的请求完成状态，并在监测到请求完成后恢复上下文。单个轮询线程通过占用CPU资源，在重负载下依然能够高效地监测请求完成状态。但是如果在引擎加载过程中创建轮询线程，每一个应用进程都会需要一个轮询线程去查询请求完成状态，那在多个加解密请求同时存在时，多个轮询线程同时工作会带来难以接受的CPU开销。

### 构造和模拟请求
为了能够提供于主流开源加解密算法库相同的接口给客户机应用程序能够使其方便地从异构加解密资源中受益，如何在半虚拟化前端像主流开源加解密算法库一样狗仔和模拟加解密请求。

OpenSSL天然为物理硬件设备提供了可供集成的engine机制，将半虚拟化前端按照对应的格式注册到OpenSSL引擎中，就能够直接像上层对接客户应用程序。但客户机应用程序在调用加解密请求时只会下发包括加解密算法、加解密密钥大小等在内的加解密元数据，作为OpenSSL引擎库的一员，半虚拟化前端需要能够根据加解密元数据构造和模拟加解密请求。

> 加解密场景中的虚拟化方案
> virtio-crypto
> 框架的半虚拟化的前端是virtio-crypto的内核驱动，通过于Linux的加解密内核框架LKCF集成，该半虚拟化前端驱动能够为客户机应用程序提供标准化的加解密服务，框架的半虚拟化藏段运行再VMM上。

## 设计
+ vcrypto-host(vcrypto-backend)
+ hypervisor
+ virtio cryptodev driver
+ vcrypto-engine
  + vcrypto-engine-frontend
  + vcrypto-engine-backend

### vcrypto-host
半虚拟化框架的后端，主要包括五个功能
1. 总体上
   + 控制方面: 基于vhost-user协议与半虚拟化前端驱动virtio vcryptodev driver共同协商协议，完成内存拓扑共享，虚拟设备能力汇报等
   + 数据面方面: 在半虚拟化前端vcrypto-engine的共享内存中为每个虚拟加解密设备都维护一个数据面队列dataqueue用于传输表示完整加解密请求的crypto请求
   + crypto请求处理: 根据加解密请求中的会话id获取到对应的会话对象，以零拷贝的方式构建本地加解密请求交由底层异构加解密资源处理
2. 对于每一个来自QEMU hypervisor的craete-session请求，vcrypto-host都会在本地创建一个会话对象，为其建立会话对象和会话ID的映射关系，并将会话ID返回给QEMU hypervisor
3. 通过构建资源池的方式管理异构加解密资源，资源池中的每一个资源组都具有单独为一个虚拟机提供加解密请求服务的能力，可以扮演一个虚拟设备资源的角色
4. 为异构加解密资源抽象成的资源组提供调度驱动层，从而调度多个异构加解密资源相互写作共同服务于同一个虚拟加解密设备，调度驱动层还允许将特定算法的请求绑定到相应的物理硬件设备上，并提供round-robin的调度测率保证负载均衡，同时提供fail-over测率提供故障转移方法
5. 供顺序排序层，对于要求严格支持FIFO测率的客户机，根据当初下发加解密请求卸载给物理硬件设备的顺序将加密后的结果依次返回给上层应用，防止由于物理硬件之间相互协作导致的乱序现象，从而避免给客户机的应用程序造成混乱

#### 总体设计
基于I/O半虚拟化技术，采用vhost-user协议作为vcrypto框架的底座。

vcrypto-host由此和virtio cryptodev driver交互，完成从半虚拟化后端到前端的虚拟设备能力汇报以及半虚拟化前端到后端的内存拓扑汇报。一次crypto加解密请求的完整处理过程需要由半虚拟化前端vcrypto-engine发起，控制面请求经由virtio cryptodev drvier和QWEMU Hypervisor、数据面请求仅经由QEMU Hypervisor到达半虚拟化后端vcrypto-host，vcrypto-host将请求下发给物理硬件设备，并在物理硬件设备处理完请求后将处理完成的请求最终返回到vcrypto-engine。

+ 设计上需要考虑三个方面的内容:
  + 如何与QEMU Hypervisor通信处理控制面请求
  + 如何与vcrypto-engine通信处理数据面请求
  + 如何与物理硬件设备通信处理并收回请求

##### 如何与QEMU Hypervisor通信处理控制面请求
QEMU Hypervisor运行在宿主机的用户态进程中，vcrypto-host运行在独立于QEMU Hypervisor的用户态进程中，但是要能够接收来在QEMU Hypervisor的控制面请求并作出响应。

+ 以需要创建会话对象时的控制面请求为例，具体可分为三个步骤
  + 运行在虚拟机中的vcrypto-engine将表示需要创建会话对象的session会话请求添加到控制面队列contrl queue随即通过VM-Exit陷入QEMU Hypervisor,由于QEMU Hypervisor将session会话请求转发给vcrypto-host
  + vcrypto-host根据从session会话请求中解析出的密钥等加解密元山上，在本地进程创建会话对象并为其分配会话ID，建立起两者的映射关系后将会话ID返回给QEMU Hypervisor
  + QEMU Hypervisor将收到的会话ID通过session会话请求反抗者给vcrypto engine

vcrypto-host需要在本地创建会话对象是因为一个会话对象会涉及多个加解密请求，仅由vcrypto-engine管理会话对象无法合理管理其生命周期，因此作为半虚拟化后端的vcrypto-host需要管理该元数据。

##### 如何与vcrypto-engine通信处理数据面请求
vcrypto-host与vcrypto-engine在数据面请求上的通信涉及到半虚拟化框架前后端通信。

+ 以下发crypto加解密请求为例，具体可分为两个步骤
  + 运行在虚拟机中的vcrypto-engine将crypto加解密请求添加到数据面队列data queue，该过程不会坏过QEMU Hypervisor
  + vcrypto-host根据crypto加解密请求间接就oqudc会话对象，根据数据面队列data queue中crypto加解密请求的内容获取到就解密数据，构造本地crypto加解密请求下发给物理硬件设备

vcrypto-host需要在本地构造crypto加解密请求的原因同样是为了合理管理其生命周期。

vcrypto-host与vcrypto-engine通信处理数据面请求的涉及关键在于半虚拟化前后端的数据面队列。

通过优化半虚拟化前后端的数据面请求通信，vcrypto-host能够以零拷贝的方式完整地获取到虚拟机中的crypto加解密请求。

##### 如何与物理硬件设备通信并收回请求
+ 单一的物理硬件设备在肚子处理加解密请求时通常会有功能有限、性能有限、鲁棒性有限的缺点，vcrypto-host允许各个异构加解密资源将完整的物理硬件设备功能划分为更细力度的资源组从而构建出资源池，因此能够实现多个物理硬件设备共同支持一个虚拟设备，并以合适的调度测率进行调度
+ 多个物理硬件设备协同工作可能导致crypto加解密请求的完成顺序不同于下发顺序，vcrypto-host通过顺序排序层保障crypto加解密请求的乱序执行现象不会被客户机应用程序察觉。

#### 会话表单设计
应用程序在执行加解密技术时通常会分为两个阶段，在第一个阶段通过包括加解密算法、加解密密钥大仙等在内的加解密元数据构建加解密会话，在第二个阶段根据加解密会话中提供的加解密元数据依次对一连串待加解密数据块进行加解密，于此同时，加解密会话还会记录加解密执行是否成功、加解密执行的成功次数等状态信息。由于一系列连续的加解密请求都会使用相同的加解密元数据，包含加解密算法类型、加密密钥、认证密钥等，vcyrpto框架抽象出会话对象来避免重复传输。

+ 创建会话的步骤
  + 当vcrypto-engine前端申请会话对象时，vcrypto-engine后端根据加解密元数据初始化一个会话ID待填充的会话对象，构造sess会话请求添加到控制面队列control queue中，并触发VM-Exit陷入到QEMU Hypervisor
  + QEMU Hypervisor在分析VM-Exit的原因后，取出加解密元数据构造出create_sess消息，并通过实现建立的unix socket连接发送给vcrypto-host
  + vcrypto-host根据加解密元数据创建本地会话对象，将该本地会话对象添加到本地会话表单中并生成会话ID，并通过unix socket连接将会话ID发送给QEMU Hypervisor
  + QEMU Hypervisor将会话ID填充到控制面队列ctrl queue的session会话请求中，触发VM-Enter恢复到虚拟机中
  + vcrypto-engine后端通过轮询从控制面队列ctrl queue中获取到会话ID，填充到会话对象后返回给vcrypto-engine前端
+ 在上面的“会话对象的创建”中涉及了两个会话对象
  + vcrypto-engine创建的会话对象
  + vcrypto-host创建的对象
+ 两个会话对象在数据结构上是相同的，但前者面向虚拟加解密设备，后者面向异构加解密设备

vcrypto-host在本地维护了基于Map数据结构的会话表单，将加解密元数据哈希得到的MD5值作为记录会话ID的key，将会话对象作为value。在vcrypto-engine希望获取会话对象时，虚拟机通过VM-Exit陷入宿主机，vcrypto-host接受来自QEMU Hypervisor的创建会话请求，创建本地会话对象，将其添加至维护“会话ID->会话对象”映射关系的会话表单，并将新增的会话ID返回给QEMU Hypervisor。在处理vcrypto-engine下发的crypto加解密请求时，vcrypto-host从数据面队列data queue中获取到crypto加解密请求，根据间接索引获取到会话ID，从本地会话表单中搜索得到对应的会话对象，并于crypto加解密请求中的加解密数据组合成本地crypto加解密请求下发给异构加解密设备。vcrypto-host维护会话表单，以相同且唯一的会话ID作为标识为半虚拟化前后端的会话对象建立索引，使得在一次加解密请求的完整执行过程中都能通过会话对象获取到相同的加解密元数据，在半虚拟化前后端维护了加解密会话状态的一致性

#### 半虚拟化前后端控制面队列和数据面队列的设计
virtqueue是一种环形缓冲队列，由描述符列表、可用环形队列表和已用环形队列表三个部分组成。

其中，描述符表中放置了真正的报文数据；在发送保温时，半虚拟化前端会添加完整报文到可用环形队列中，半虚拟化后端从可用环形队列中获取后将报文添加到已用环形队列中；在接收报文时，半虚拟化前端会添加空白报文到可用环形队列中，半虚拟化后端从可用环形队列中获取并填充后将报文添加到已用环形队列中。

+ 为了实现与QEMU Hypervisor和vcrypto-engine的通信，vcrypto-host为每个虚拟加解密设备维护了一个控制面队列control queue和数据面队列data queue，两者都通过virtqueue实现，同时还会维护一个unix socket文件描述符。
  + 控制面队列contrl queue用于在virtio cryptodev driver和QEMU Hypervisor之间传输包含加解密元数据的session会话请求，QEMU Hypervisor将session会话请求转发给vcrypto-host，等待vcrypto-host在奔鼎进程创建会话对象并返回会话ID
  + 数据面队列data queue用于在vcrypto-host和virtio cryptodev driver之间传输包含完整加解密数据和加解密元数据的crypto加解密请求；
    + 在下发加解密请求时，vcrypto-host通过间接索引只会跑从数据面队列data queue中得到加解密元数据和加解密数据，构造本地crypto加解密请求后下发给物理硬件设备
    + 在返回已处理完成的加解密请求时，vcrypto-host从物理硬件设备取回已处理完成的crypto加解密请求，设置状态位并添加到数据面队列data queue
  + unix socket文件描述符用于在vcrypto-host和QEMU Hypervisor之间建立通信链路，通过socket接口实现基于vhost-user协议的消息传输
+ 其中，控制面请求会经过QEMU Hypervisor，而数据面请求则不会

#### 资源池设计
vcrypto-host对异构加解密资源的管理方式是构建资源池，允许哥哥异构加解密资源将完整的物理硬件设备功能划分为更细粒度的资源组，从而实现多个异构加解密资源共同支持一个虚拟设备。资源池中的最小粒度为资源组，每一个资源组都应当具有单独为一个虚拟机提供加解密请求加速服务的能力，即每一个资源组都可以扮演一个虚拟设备资源的角色，只要异构加解密资源对于资源组的划分能够满足该要求即可。

#### 调度驱动层设计
一方面为了统一对上层暴露的使用接口，另一方面为了集成对底层多种异构加解密资源的调度能力，vcrypto-host设计了调度驱动层，通过提供多种合适的调度测率，针对性地客服单一异构加解密资源肚子处理加解密请求的功能有限、性能有限、鲁棒性有限的缺点
+ 克服单一异构加解密资源功能有限的缺点时，调度驱动层以扩展虚拟设备功能为目标，基于算法匹配的调度测率，在接收到vcrypto-host下发的crypto加解密请求时，通过解析加解密元数据获取到请求期望使用的加解密速发，并以此为依据将请求分别调度给提供或优化了该加解密算法的异构加解密资源，使得不同的异构加解密资源都能够发挥自己的优势
+ 在客服单一异构加解密资源性能有限的缺点时，调度驱动层以提升虚拟设备性能为目标，基于round-robin的调度测率将vcrypto-host下发的crypto加解密请求轮转的批量下发给不同的异构加解密资源，在充分利用异构加解密资源能力的同时，通过负载均衡使得整体的加解密性能不会被单一异构加解密资源所限制
+ 在克服单一异构加解密资源鲁棒性有限的缺点时，调度驱动层以增强虚拟设备鲁棒性为目标，基于fail-over的调度测率将资源组划分为主资源组和副资源组，在接收到vcrypto-host下发的crypto加解密请求时，首先尝试下发给主资源组，若出现请求下发失败的情况，则重新将失败的部分下发给副资源组，从而提高为虚拟设备提供服务的鲁棒性。

#### 顺序排序层设计
+ 调度驱动层也可能带来加解密请求乱序执行的问题，及加解密请求的完成顺序不同于下发顺序。
  + 不同的异构加解密资源的实现细节不同，处理加解密请求的速度存在差异，处理速度较快的异构加解密资源即使较晚收到下发的加解密请求也可能更早完成
  + 在使用调度驱动层fail-over调度测率的情况下，异构加解密资源出现故障时，调度驱动层会选择其他的异构加解密资源进行重传，从而导致较早下发的加解密请求更晚完成。
+ 客户机应用程序下发的相邻加解密请求前后可能存在逻辑上的关系，或者两个加解密请求的待处理数据块实际上是由一个更大的待处理数据块切分而成的，加解密请求乱序执行的结果如果直接返回显然会造成客户机应用程序的混乱
+ 解决方法
  + 在vcrypto-engine中根据地址代替顺序来标识加解密请求
  + 在vcrypto-host实现顺序排序与层，强制保证调度驱动层返回FIFO的请求完成顺序


### vcrypto-engine
半虚拟化框架的前端，主要功能是将虚拟设备提供的加解密能力集成为专用的OpenSSL引擎提供给客户机应用程序。

+ 需要实现的功能
  + 能够被客户机应用程序出通过通用的接口对虚拟加解密设备的引擎进行加载并执行初始化流程
  + 需要接收来自应用进程的加解密元数据并构造session会话请求，通过控制面队列contrl queue下发给 QEMU Hypervisor获取到会话对象
  + 需要接收来自应用进程的加解密数据并构造crypto加解密请求，通过数据面队列data queue下发给vcrypto-host
  + 需要监测crypto加解密请求的完成状态，在请求完成后批量取回并通知唤醒应用进程
+ 解决的挑战
  + 在vcrypto-engine后端缓存加解密请求累计到一定数量后一起批处理下发，减轻半虚拟化前端驱动压力的同时提高异构加解密设备的利用率，
  + 通过将vcrypto-engine划分为前后端架构的设计，使得轮询线程的开销局限在作为守护进程的vcrypto-engine后端上，避免了在每个应用进程上都创建轮询线程带来的高昂CPU开销
  + 通过将vcrypto-engine前端实现为动态链接库并集成到OpenSSL中，使得客户机应用程序能够方便地从异构加解密资源中受益

vcrypto-engine支持通过异步模式处理加解密请求，并在涉及上采用了前后端架构，可以分为engine-frontend和engine-backend两个模块

#### vcrypto-engine-frontend
被实现为一个动态链接库，在呈现给用户进程的形式上与常见的OpenSSL引擎保持一致，应用进程可以通过OpenSSL API对引擎进行加载，并像客户机应用程序提供对应的加解密算法支持

#### vcrypto-engine-backend
+ 被实现为一个守护进程，主要有三个功能
  +  接受来自frontend传入的加解密元数据，通过virtio cryptodev driver提供的接口将session请求添加到控制面队列contrl queue以完成会话对象的创建
  +  遍历位于engine前后端共享内存中的数据传输队列trans queue获取来自engine前端的crypto请求，经过缓存和批处理后通过virtio cryptodev driver提供的接口一次性添加到数据面队列data queue中完成请求下发
  +  通过异步模式处理加解密请求时，engine backend负责监测加解密请求的完成状态，对于已经处理完成的加解密请求则会像对应的进程发送通知

#### 总体设计
采用vhost-user协议作为vcrypto框架的底座。客户机应用程序在不对自身做任何修改的情况下通过通用的接口加载vcrypto-engine完成初始化，然后依次下发加解密元数据和加解密数据给vcrypto-engine，持续不断地进行加解密请求的处理。vcrypto-engine作为半虚拟化前端，设计到与客户机应用程序的直接交互，需要提供尽可能有些的加解密请求下发和处理能力。
+ 通过异步模式处理加解密请求以提供高性能和鲁棒性
+ 通过轮询模式和划分前后端降低CPU资源的开销

### QEMU hypervisor
虚拟机监控程序，最常见的就是QEMU Hypervisor，主要有三个功能
+ 为虚拟机提供诸如虚拟机内存等基本资源
+ 在虚拟机启动的过程中，QEMU Hypervisor通过VM-Exit拦截虚拟机操作系统内核的PCI总线访问行为，并为虚拟机模拟出一个符合virtio表春的虚拟加解密设备
+ 在虚拟加解密设备的启动和执行过程中，QEMU Hypervisor依然可以通过VM-Exit辅助半虚拟化前端驱动virtio cryptodev driver与半虚拟化后端vcrypto-host进行通信

QEMU Hypervisor与半虚拟化前端vcrypto-engine的共享内存中为每个加解密设备都维护一个控制队列ctrl queue用于传输表示加解密元数据的session请求，将session请求转发给vcrypto-host，并将从vcrypto-host获取的会话ID返回给vcrypto-engine，完成会话对象的创建和协调

### virtio cryptodev driver
运行在虚拟机中，作为用户态半虚拟化前端驱动，主要功能是在虚拟机完成启动后，代替虚拟机操作系统内核接管虚拟机加解密设备，并向上层暴露统一的接口。

virtio cryptodev driver能够提供的接口包括初始化虚拟加解密设备的接口、设备启动和设备配置的接口、将session请求添加到控制面队列ctrl queue的接口、将crypto请求添加到数据面队列data queue的接口、将已经处理完成的crypto请求取出的接口等

