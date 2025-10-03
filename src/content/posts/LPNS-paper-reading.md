---
title: LPNS-paper-reading
published: 2025-07-16
description: '论文LPNS: Scalable and Latency-Predictable Local Storage Virtualization for Unpredictable NVMe SSDs in Clouds的阅读笔记'
image: ''
tags: ["linux", "virtualization"]
category: notes
draft: false 
lang: ''
---
# 论文整理
## abstract
+ latency在一些情况下很重要,但是过往研究主要集中关注如何提高虚拟机的吞吐量,而没有重点关注延迟
+ LPNS关注的是本地存储,不考虑网络存储
  + self-feedback control
  + flexible IO queue
  + command scheduling
  + scalable polling design
  + deterministic network calculus-based formalization method(这个是一个性能评估的算法)
+ 主要竞品: MDev-NVMe和SPDK

## Introduction
+ 云服务器厂商提升他们的云服务器的IO吞吐量,并且还保证每个客户的带宽公平, 但是忽略了有的客户可能不需要很高的吞吐量,但是要保证他们的IO延迟不能过高
+ 在现在的方案下,如果一块硬盘被多个客户机共享,那么可能延迟敏感的客户,在其它客户进行大量读写的时候,导致他自己的延迟陡增,不能满足他的需求
  + 原因: 现代SSD控制器在接收到大量IO请求的时候,是以轮转调度的方式来处理的,所以当其它客户大量发IO请求的时候,延迟敏感的客户就被拖下水了
+ LPNS干了什么
  + 在传统的虚拟机和硬件之间夹进去了一个动态队列调度器
  + IO command调度器,保证延迟敏感客户不会被其它普通用户拖慢
  + 提出通过deterministic network calcules的方法的方法来从理论上计算细腻话存储中延迟的可预测性,给出确定的延迟上界
+ 创新/贡献
  + 提出使用确定性网络演算,来为延迟上界提供数学模型
  + LPNS的架构
    + 自适应反馈机制:根据多租户负载动态调整资源分配
  + 基于中介式透传
    + 云服务器厂商不需要更新硬件/花钱
  + 实验结果确认有效果

## background
### NVMe
NVMe在接受IO请求的时候是怎么做的

HostOS构造NVMe指令,然后放到submission queue中,并告知NVMe控制器.NVMe控制器执行命令,把数据拷贝到DMA中,然后构造完成结果到completion queue中

### Local NVMe Storage for Clous Services
+ 网络存储(存储池)
+ 存储在本机中(和CPU、内存在同一个物理机器中)

- 网络存储的问题
  - 对存储对读写性能会受到网络的影响(如其它的网络请求可能会影响存储的IO性能)
  - 云服务器厂商还得购买专门的设备提供网络存储的服务
  - 用户可能希望能够使用baremetal机来自定义,这种时候一般都一定是本地存储

### Deterministic Network Calculus
+ 本来是用来计算在最坏情况下网络通信延迟
+ 重要的三个概念
  + arrival curve: 输入的流量在任意时间窗口内的最大积累量
  + service curve: 系统在给定时间窗口内保证提供的最小服务量
  + virtual delay: 两个curve的最大水平距离
+ 针对NVMe进行的一些假设
  + NVMe的处理性能稳定不变,除非在GC并且使用FIFO策略的时候
  + 多租户情况下,需要吞吐量的用户使用最大的队列深度; 需要延迟敏感的租户队列深度为1
+ 三个概念转换到NVMe上
  + arrival curve: 虚拟机的IOPS,决定SSD会受到的IO请求频率
  + service curve: SSD至少能够保证的IOPS
  + virtual delay: 关注的IO延迟

## motivation
+ 过去虚拟化方向,都只注重吞吐量,不注重延迟
+ K2针对延迟,但是吞吐量下降太多,而且不是针对虚拟化NVMe的

## design and immplementation
### overview
+ 基础要求
  + 延迟可预测(不高于设定值)
  + 性能损失不能太多(5%),严格保持延迟要求
  + 不同虚拟机的延迟在初始化的时候确定,运行时更改要经用户同意
+ 架构:
  + 有点类似“劫持”来自虚拟机的IO请求,然后根据要求来重新排队,确保延迟敏感的用户的请求始终能够尽快发送到NVMe上
+ 全虚拟化
  + hypervisor直接把HWQ的寄存器映射到VM地址空间,通过IOMMU限制VM只能访问分配给它的HWQ,同时拦截危险命令.
  + 对于虚拟机而言,它感觉就是在直接通过标准的NVME方式发IO命令,不用像VirtIO一样修改虚拟机的驱动
+ 自反馈的QoS控制
  + 在创建虚拟机的时候给这个虚拟机一个标签,判断这个虚拟机是否需要延迟敏感保证
  + 在轮训线程中插入探针,定期收集各种数据指标,这些指标可以为调度器提供决策依据,来动态调度
+ 灵活可扩展到轮询
  + 可以用一个轮询线程轮询所有的延迟不敏感的虚拟机,并充分利用NVMe的性能
  + 对所有延迟敏感的虚拟机都专开一个轮询线程,确保延迟
  + 以上两种线程都根据运行时状态动态开启或关闭,需要的时候开启保证性能,不需要的时候关闭减少CPU负担
+ 硬件队列池
  + 1-1 HWQ: 用于延迟敏感虚拟机
  + 1-N HWQ: 用于延迟不敏感虚拟机
  + 全局队列调度器动态的吧1-1 HWQ和1-N HWQ分配给需要的虚拟机,保证延迟
+ IO命令的控制
  + 通过控制(暂时不发放)部分IO请求,保证延迟敏感的请求能够一定在时间内完成
  + 控制是根据deterministic network calculus的限制来调度的


### 可扩展的IO队列处理
+ 过去的方法: 静态的队列,SSD暴露出来HWQ的数量直接限制了可以共享一块SSD的VM的数量
+ LPNS: 灵活的重映射,虽然总量仍然不可以超过HWQ的总量.
+ design: IO queue scheduler, 分时复用HWQ,把所有的HWQ放到HWQ Pool中
  + 1-1HWQ: 一个HWQ只绑定到一个Virtual Queue上,
  + 1-NHWQ: 一个HWQ同时接受来自多个VQ的请求
  + 在初始化LPNS的时候就必须要确定1-1HWQ和1-NHWQ的数量
  + 1-1HWQ一般分给SVM,1-N就分给IVM
+ 调度算法/策略
  + 输入:
    + 各VM的QoS目标
    + 运行时的工作负载
  + 阶段:
    + 分层VQ权重计算: 文强诶所有VM的VQ动态分配权重
    + 切换HWQ:高优先队列绑定到1-1HWQ上,剩下的绑定去1-NHWQ上
  + 权重的计算规则:
    + SVM: 如果VQ为空,权重为0,否则为最大权重
    + IVM: 权重=积压命令数, 负载更重的可以相对获得更高的优先级
  + sleep直到下一次调度
+ 潜在问题: 当切换HWQ和VQ的绑定的时候,可能上一次VQ的请求并未被完成
  + 直接转发的问题: IO error
  + 解决方法: seamless switching mechanism
    + 扩展虚拟nvme命令,增加一个VM的表示字段,这样的话HWQ就可以同时处理来自不同VM的VQ命令

### IO命令限流机制
通过deterministic network calculus的来解决多个虚拟设备竞争同一个SSD的问题,这样hypervisor就可以通过在不同的虚拟设备之间调度资源队列+对IO命令限流,来同时消除OS级别和设备级别的延迟干扰

+ arrival curve:
  + $\theta$: 延迟不敏感虚拟机比延迟敏感虚拟机IO命令数量多出来的倍数
  + $j$: 延迟不敏感虚拟机数量
  + $i$: 延迟敏感虚拟机数量
  + $p$: IOPS
  + $d$: 队列深度
  + $v = p \cdot (j \cdot \theta + i)$: IO命令的提交频率
  + $b = d \cdot (j \cdot \theta + i)$: 一次性还可以发送的命令的数量
  + 到达曲线: $\alpha(t) = v \cdot t + b$
+ service curve:
  + $R$: 硬件并行(读)写速率 (因为一般随机读读性能高于写,采用最低值,所以是写)
  + $L_h$: 完成一个IO请求的最低延迟
  + 服务曲线$\beta(t) = R \cdot t + L_h$
+ Latency Upper bound
  + 公式: $d(t) = inf {\tau >= 0 | \alpha(t) <= \beta(t + \tau)}$
  + $L_{max} <= \frac{b}{R} + L_h = d \cdot \frac{(j \cdot \theta + i)}{R} + L_h$

+ LPNS通过控制$\Omega = j \cdot \theta + i$来控制可预测延迟表现
+ 比较小的$\Omega$可以确保更加可控,但是吞吐量损失也较多
+ LPNS允许VM提前预设好租户定义的延迟目标,来调整$\Omega$参数

### 可预测延迟的IO处理
#### SVM的处理
1. VM产生IO请求,LPNS的shadow IO queue会存储到SQ中
2. 轮询线程立刻获取这个SQ的head,并且翻译DMA和LBA地址,并存放到1-1HWQ中,通过doorbell register告知硬件
3. 硬件收到请求以后,获取数据,通过DMA把数据放到内存中,然后把完成消息放入HWQ的CQ中
4. 轮询线程发现CQ有东西了,立刻把CQ添加到shadow queue的CQ中,并且向VM发送一条中断告知它数据读完了

#### IVM的处理
当数据进入shadow queue,发送到HWQ之前,有可能会被轮询线程限制,以防止感染延迟敏感虚拟机的性能.

### 讨论
+ 轮询的效率: 1个SVM,多个IVM的时候,LPNS可以充分利用测试设备的性能,并且比竞品有更好的可扩展性
+ 性能损耗: 使用two-phase数组,一个phase被读并做计算,另一个phase用来写入IO数据. 没有锁. 可以忽略这点性能损耗
+ 资源损耗: 如果不好好处理,那么数以万计的IO请求可能会消耗几百KB的空间
  + 分段检测:
    + 10ms做一个检测(远短于调度周期)
    + 计算平均指令的延迟
    + 获得周期的结果
  + 轮询线程可以动态被调整为空闲状态
    + 如果500ms没有IO,轮询线程转空闲
    + 遇到IO,立刻激活轮询线程
+ 不足与改进:
  + 要针对特定场景调优
  + 当写性能远低于读性能的设备上时,吞吐量的牺牲较大

## 实验
实验主要有两个部分,一方面是比较与其它主流SSD虚拟化的性能差异; 另一方面就是比较QoS控制系统的优劣
### 实验准备
+ 硬件的详细型号
+ 软件的详细情况
  + 保证总的HWQ数量小于总的VQ数量,来模拟在资源短缺的场景下的环境
+ 负载: 使用FIO
  + 引擎使用libaio(非阻塞异步io),能够绕过内核缓冲区直接访问磁盘

### micro benchmark
+ 1个SVM
+ 多个IVM
+ 依次测试LPNS, MDev-NVMe, SPDK with vhost-blk, virtio
+ SVM最轻量的负载
+ IVM负载刚开始和SVM一样,然后主线提高负载
+ 整体上LPNS效果好,但是在PM1735上,因为读写性能不一致,所以IVM的吞吐率牺牲比较多



# YouTube
## background
+ High-performance NVMe Storage
  + Efficient and scalable interface designed for high-performance SSDs
  + up to 65535 pairs of I/O queues, up to 65535 depth, enabling high-parallel I/o processing
  + High-parallel SQ/CQ interacation between the host and tghe SSD controller(with interrupt)
  + High throughtput and micro-second-level latency advantages over the traditional interfaces, such as sata
+ two solutions for cloud instance storage resources
  + using remote storage pools or dedicated storage servers
    + disaggregated computing and Storgae resources
    + Efficient, scalable and portable
    + Related works: Reflex, LeapIO, Mellonox SNAP
  + using the local storage resources from the SSDs on the native servers
    + removing the additional bottleneck of latency performance incurred by the network
    + removing expensive costs on network devices of cloud infrastructures
    + more friendly to bare-metal cloud service tenents
    + related works: VFIO, VirtIO, SPDK, MDev-NVMe, VirtIO-fs, SR-IOV, FVM, BM-store

## motivation
+ limitation of the current storage virtualization
  + only concentrating on high throughput, no efficient latency-predicatble QoS control
  + inaccurate perception of the imbalanced multi-tenant workloads with different QoS requirement
  + cannot bypass the device-sdie latency interference on the OS level
+ device-side latency interference
  + multiple VMs with virtulized NVMe storage sharing the same underlying NVMe SSDs
  + throughtput-intensive VMs complete for NVMe I/O queue resources, congestion on SSD controller
  + latency degradation, unbounded latency, and the miss  of latency QoS (or SLA)
+ typical case
  + VM1 run light but latency-sensitive adata I/O workloads (e.g. data query) with a 30 us SLA
  + VM2 run thourhgput-intensive data I/O workloads without latency QoS requirement but arriving with burst
+ further strudy in the device-side latency interference
  + the state-of-the-art NVMe virtualization (SPDK, SR-IOV, MDev-NVMe) cannot eliminate the latency innterference
  + the more intensive the competitor's workload is, the more severe latency interference happens
  + average latency on the NVMe controller grows from 62.5% to 93.0% of the total latency with the increasing IOPS of othe VM2 workload: the controller of unpredicatble NVMe SSD is bottleneck
  + in HW/SW co-design virtualization, an accelerator card (smartNIC or FPGA) attacehs standard and unpredictable NVMe and use SR-IOV, cannot bypass the device-sidew latency interference
+ Storage I/O QoS researches (with or without virtualized device designs)
  + High total throughput: SPDK, Spool, LeapIO, FVM, BM-Store2
  + Fair queueing control: FIOS, FLIN, MQFQ, D2FQ
  + latency control: differentiated storage services, autossd, K2

## design
+ Scalable architecture
  + Hybrid depolyment of host processes, containeres, VMs
+ full virtualization
  + each virtualized storage is a device with NVMe feature(/dev/nvmeXnY)
+ self-feedback Qos control
  + SVM: VM with latency-predictable QoS
  + IVM: VM without latency-predictable QoS
+ flexible and scalable polling
  + Accelerating VM I/O processing
  + flexibly configure the number of threads
  + one thread can utilize the high throughput
  + one dedicated polling thread for each SVM
+ scalable I/O queue handling
  + design a hardware queue pool in the kernel module to manage queue resources
  + flexibly allocate any number of the HWQs(<the max number of NVMe SSD) into the Pool
  + An I/O queue scheduler to manage the Time Division Multiplexing(TDM) of I/O queues
  + 1-1 HWQs (bound to 1 VQ) and 1-N HWQs (shared by multiple VQ)
+ I/O conmmand throttling with DNC
  + overcom device-level latency interference from OS-level storage virtualization
  + the polling thread can monitor the workload and control the command distribution speed
  + LPNS Module can control the throttling IVM throughput as $\theta$ times of IOPS of SVM with the slowest submission rate
  + a deterministic queuing system with deterministic network calculus
+ using the deterministic network calculus to modle NVMe virtualization
  + Concentraing point: 4K block size random read and write, the device-level latency interference are caused by the congestion on the SSD controller
  + scenario: j iVMs co-running with i SVM, the IOPS of SVM is p and its command queue depth is d
  + arrive curve: $\alpha (t) = v \cdot t + b = (p \cdot (j \cdot \theta + i)) \cdot t + d \cdot(j \cdot \theta + i)$
  + service curve: $\beta (t) = R \cdot t + L_h$
  + latency upper bound $L_{max} <= \frac{b}{R} + L_h = \frac{d \cdot(j \cdot \theta + i)}{R} + L_h$
+ for SVM 
  + using 1-1 HWQ
  + dedicated polling thread for better latency performance
  + no I/O command throttling
  + no throughput sacrifice
+ for IVM
  + using the idle 1-1 HWQs and sharing 1-N HWQs
  + one polling thread for all IVMs, reducing overhead
  + dynamic I/O command throttling according to DNC
  + smaller throughput sacrifice than previous works in K2

## implementation
+ all the implementations of LPNS are included in a linux kernel source code
+ LPNS implementation is based on mediated pass-through
  + cooperate by the vfio-0mdev support of linux kernel
  + a kernel module (nvme-mdev.ko) to provide virtualized storage with full NVMe
  + coexist and cooperate with the origianl nvme.ko module (the NVMe driver)
  + Admin and I/O queue resource management (resource emulation and pass-through)
  + polling thread is resposible for performance detection, optimization and I/O throttling
  + providing CONFIG_NVME_MDEV kconfig for cloud system maintainers
+ LPNS works on the mainstream intel/AMD CPUs and all commercial NVMe SSD

## evaluation
+ hardware comfiguration
+ Micro benchmark (comparison with other storage virtualization)
  + Flexible I/O tester(FIO), 4K random read or write, using libaio and O_DIRECT
  + VirtIO, SPDK vhost-blk, MDev-NVMe on intel P5800X, MDev-NVMe and SR-IOV on samsung PM1735
+ application benchmark
  + webuser
  + ycsb

## conclusion
+ latency predictability trade-off
  + different NVMe SSD has different service curve
  + most NVMe SSDs has poorer random write performance thant read, wasting more total throughput than random read
+ performance and resource overhead
  + flexible and scalable polling to reduce CPU overhead
  + periodical workload and performance detection, saving kernel memory resoures
+ LPNS: Scalable and Latency-Predicatble virtualization
  + we analyze the device-level latency interference, and argu the signifigance of overcoming this interference from ths OS-level storage virtualization
  + we design LPNS, the first OS-level NVMe virtualization solution with latency-predictable QoS enhancement
  + we implement LPNS in the linux kernel without any further purchase of accelerator cards