---
title: asic-based-compression-accelerator
published: 2026-05-14
description: '论文asic based compression accelerator for storage system阅读笔记'
image: ''
tags: ["linux", "compression"]
category: notes
draft: false 
lang: ''
---
# asic-based compression accelerators for storage systems paper reading

## abstract

这篇论文讨论的是面向存储系统的硬件压缩加速器，尤其关注一个核心问题：压缩加速器究竟应该放在哪里？

论文比较了三类主流 CDPU（Compression/Decompression Processing Unit）放置方式：

- Peripheral CDPU：作为 PCIe 外设卡接入，例如 Intel QAT 8970
- On-chip CDPU：集成在 CPU 内部或 CPU die 附近，例如 Intel QAT 4xxx
- In-storage CDPU：集成在 SSD 控制器内部，例如本文提出的 DPZip / DP-CSD

论文提出了一个 in-storage ASIC 压缩加速器 DPZip，并将其集成到商业 PCIe 5.0 SSD 中，形成 DP-CSD。DPZip 主要面向 SSD 的 4KB page 进行压缩，使用硬件友好的 LZ77、Huffman 和 FSE 编码设计，在资源受限的 SSD 控制器中实现较高吞吐、较低延迟和较好的压缩率。

这篇文章最有价值的地方不是简单证明“硬件压缩比软件快”，而是系统性地说明：硬件压缩加速器的性能不能只看单点吞吐，还必须结合放置位置、数据搬运路径、应用可见性、系统功耗、多设备扩展性和多租户隔离性一起分析。

一句话总结：

硬件压缩加速器的关键问题不是“能不能加速压缩”，而是“压缩发生在系统栈的哪个位置，以及这个位置会如何影响整个存储系统”。

## introduction

在云数据中心中，数据压缩是一个非常重要的技术。压缩可以减少存储空间、降低数据传输量、减少存储设备写入量，并进一步降低成本和能耗。

但是，压缩本身并不是免费的。Deflate、Zstd 这类压缩率较好的算法通常需要较多 CPU 资源。相对地，Snappy、LZ4 这类轻量压缩算法速度快，但是压缩率较低。对于大规模数据中心来说，哪怕压缩只占总 CPU 周期的一小部分，也会被放大成非常可观的资源消耗。

因此，硬件压缩加速器开始变得重要。它的目标是：

- 保持接近 Deflate / Zstd 的压缩率
- 降低 CPU 压缩开销
- 提升吞吐和降低延迟
- 在存储系统中降低总体成本和能耗

已有的硬件压缩加速器可以大致分为三种：

- 作为 PCIe 外设卡存在，例如 Intel QAT 8970
- 集成在 CPU 内部，例如 Intel QAT 4xxx
- 集成在 SSD 控制器内部，例如 ScaleFlux CSD 和本文的 DPZip

这篇论文认为，以往工作往往只研究单个加速器，或者使用模拟方法，缺少真实系统中多个 ASIC-based CDPU 的横向比较。尤其是 in-storage CDPU，也就是把压缩放到 SSD 内部的方案，公开研究仍然较少。

因此，本文试图回答几个问题：

- 在 SSD 控制器这样资源受限的环境中，如何设计一个 ASIC 压缩加速器？
- CDPU 放在 PCIe 外设、CPU 片上、SSD 内部，对吞吐、延迟和应用性能分别有什么影响？
- 硬件加速器的微基准性能能否直接转化为真实应用性能？
- 多设备扩展和多租户 SR-IOV 场景下，不同 CDPU 的隔离性如何？
- 从功耗和 TCO 的角度，in-storage compression 是否有优势？

## background

### compression in storage systems

存储系统中常见的无损压缩主要依赖两类技术：

- dictionary compression
- entropy coding

dictionary compression 的典型代表是 LZ77。它会在过去的数据窗口中寻找重复片段，然后把重复内容表示为类似 offset + length 的形式。

entropy coding 的典型代表是 Huffman coding、ANS、FSE 等。它的目标是让高频符号使用更短编码，让低频符号使用更长编码，从而进一步压缩数据。

Zstd 这样的现代压缩算法通常把 LZ77 和 entropy coding 结合起来。论文中的 Figure 2 分析了 Zstd 在不同 chunk size、compression level 和 data entropy 下的耗时组成。一个重要观察是：在较高压缩级别下，LZ77 的模式搜索会成为主要计算开销。

这也解释了为什么压缩适合硬件加速：LZ77 的模式匹配和 entropy coding 有较强的数据路径特征，可以被做成流水线硬件；但是压缩算法又有复杂的分支和数据依赖，直接把软件算法搬到硬件中并不现实。

### what is CDPU

CDPU 是 Compression / Decompression Processing Unit，也就是专门负责压缩和解压缩的硬件单元。

论文把 CDPU 分为三种 placement：

#### peripheral CDPU

Peripheral CDPU 是插在 PCIe 总线上的独立压缩加速卡，例如 Intel QAT 8970。

它的典型数据路径是：

应用程序或驱动准备压缩请求，然后 QAT 通过 PCIe DMA 从 host memory 读取输入数据，压缩完成后再通过 DMA 写回 host memory。

优点：

- 不占用大量 CPU 计算资源
- 可以作为独立 PCIe 卡部署
- 可以服务多个应用或多个 VM

缺点：

- 数据必须在 host memory 和 PCIe 设备之间来回移动
- PCIe DMA 和设备队列会带来额外延迟
- 部署成本和系统复杂度较高

#### on-chip CDPU

On-chip CDPU 是集成到 CPU 内部或 CPU package 中的压缩加速器，例如 Intel QAT 4xxx。

它相比 PCIe QAT 的核心优势是更靠近 CPU cache / memory hierarchy，可以利用片上互连和 DDIO / LLC 等机制降低数据访问延迟。

优点：

- 比 PCIe 外设型 QAT 延迟更低
- 更接近 CPU 和内存
- 对小块数据处理更有优势

缺点：

- 吞吐不一定比 PCIe 卡更高
- 扩展性受 CPU socket 数量限制
- 多租户共享时仍然可能发生资源争用

#### in-storage CDPU

In-storage CDPU 是集成在 SSD 控制器内部的压缩加速器，例如本文的 DPZip。

它的核心思想是：数据本来就要经过 SSD 控制器，那么可以在写入 NAND 之前直接压缩，在读取后直接解压。这样可以减少 host CPU、host DRAM、PCIe 加速器之间的数据搬运。

优点：

- 消除 host-CDPU 数据搬运
- 可以减少 NAND 写入量
- 压缩能力随 SSD 数量扩展
- 在多租户场景下可以利用 SSD 自身的 VF / queue 隔离

缺点：

- 压缩对主机和应用通常是透明的
- 上层应用不一定能利用压缩结果优化自身数据结构
- SSD FTL 需要处理压缩后变长数据带来的映射复杂性
- 算法灵活性通常比 CPU 软件差

## motivation

论文的核心 motivation 可以分成三层。

### in-storage CDPU design is underexplored

虽然已经有一些计算存储设备和带压缩功能的 SSD，但是公开研究中对 in-storage ASIC CDPU 的微架构设计、FTL 集成和系统影响讨论不足。

把压缩放到 SSD 内部并不是简单地把一个压缩模块塞进控制器。SSD 控制器有严格的面积、功耗和 SRAM 限制，还要同时处理 NVMe、FTL、ECC、NAND 管理等任务。因此，DPZip 的设计必须在压缩率、吞吐、延迟、面积和功耗之间取舍。

### placement matters

过去常见观点可能认为：新一代片上加速器一定比 PCIe 外设型更先进。但论文发现，on-chip QAT 4xxx 的主要收益是降低延迟，而不是提升吞吐。

这说明 CDPU 的 placement 不仅影响硬件本身，还影响数据路径：

- PCIe 外设型：吞吐可能不错，但端到端延迟较高
- CPU 片上型：延迟较低，但吞吐和扩展性受限
- SSD 内嵌型：最小化数据搬运，但应用层不一定感知

### microbenchmark does not equal application performance

硬件压缩在 microbenchmark 上吞吐很高，不代表真实应用一定更快。

例如 RocksDB 中，应用可见的压缩可以改变 SSTable 的逻辑大小和 LSM-tree 结构，从而影响读放大和延迟。而 in-storage compression 对应用透明，只减少 SSD 内部物理写入量，不会自动改变 RocksDB 的逻辑数据布局。

因此，硬件加速器必须放在完整系统栈中评估。

## DPZip: in-storage ASIC compression accelerator

### overview

DPZip 是本文提出的 ASIC-based in-storage compression accelerator。它被集成在商业 PCIe 5.0 SSD 控制器中，作为 DP-CSD 的核心压缩引擎。

DPZip 的基本特点：

- 运行频率约 1GHz
- 每周期处理 8 bytes
- 面向 4KB SSD page 进行压缩
- 使用 LZ77、Huffman、FSE
- 占 SSD controller die area 大约 4.5% 到 5%
- 目标吞吐可达到 16GB/s
- 4KB transfer latency 约为微秒级

DPZip 的设计目标并不是完整复刻软件 Zstd，而是实现一个适合 ASIC 的 Zstd variant。它通过硬件友好的算法设计，在有限硅面积下接近 Deflate / Zstd 的压缩率，同时保持较高吞吐和较低延迟。

### LZ77 encoder

DPZip 的 LZ77 encoder 是整个压缩路径中的核心。

软件中的 LZ77 为了追求压缩率，通常会使用较大的 sliding window 和复杂的匹配搜索。但是在 SSD 控制器中，这样做会消耗大量 SRAM 和逻辑资源，不适合 ASIC 实现。

DPZip 做了几种简化和优化：

- 使用小型 bounded hash table
- 每个 hash entry 只保存少量候选位置
- 使用 circular FIFO 淘汰旧 entry
- 先做快速 hash check，再做 byte-wise verification
- 使用 partial lazy matching
- 接受 first-fit match，避免复杂回溯

换句话说，DPZip 牺牲了一点点压缩率，换取硬件实现的确定性、低面积和高吞吐。

这里的设计取舍很典型：

- 软件压缩更关心压缩率和算法灵活性
- ASIC 压缩更关心流水线、面积、SRAM、时序和功耗

### LZ77 decoder

解压缩中的一个难点是 match copy，尤其是短 offset 的重叠复制。

例如 LZ77 解码中可能遇到类似“从刚刚写过的位置继续复制”的情况。这种 overlapping match 在软件中可以通过循环处理，但在硬件中会引入读写冲突和 pipeline stall。

DPZip 的做法包括：

- literal buffer 和 history buffer 分离
- 使用 dual-port SRAM
- 为短 offset 使用小型 register-backed recent-data buffer
- 对长 offset 使用 prefetch
- 对最近写入数据使用 bypass logic

这样可以降低 SRAM 访问延迟，并使解压吞吐更加稳定。

### Huffman and FSE

DPZip 使用 canonical Huffman coding。相比保存完整 Huffman tree，canonical Huffman 只需要保存 code length，然后可以重构 codebook。这对硬件更友好，因为它减少了树结构、指针和不规则访问。

论文中 Huffman tree canonization 被设计成三个稳定阶段：

- Leaf Scan & Cap
- Deterministic Redistribution
- Logarithmic Hole Repair

同时，DPZip 对 Huffman code length 设定上限，例如 11-bit，以限制硬件复杂度。这样做可能略微影响压缩率，但有利于时序收敛和硬件面积控制。

FSE encoder / decoder 则被设计为与 Zstd 软件实现兼容，并通过 FSM 和深流水线实现确定性延迟和高吞吐。

## DP-CSD: DPZip-powered SSD

### architecture

DP-CSD 是搭载 DPZip 的 compression-enabled SSD。

在写路径中：

- Host 通过 NVMe 发送 write command
- SSD controller 的 Queue Manager 处理 NVMe 请求
- 数据通过 DMA 进入片上 Shared Buffer Memory
- DPZip 在 SSD 内部进行压缩
- 压缩后的数据进入 flash controller
- 最终写入 NAND flash

在读路径中：

- flash controller 从 NAND 取出压缩数据
- DPZip 解压缩
- 解压后的数据通过 PCIe 返回 host

这个设计的核心是 inline compression。压缩发生在 SSD 内部 IO path 中，对 host 透明。

### FTL integration

把压缩放进 SSD 后，FTL 会变得更复杂。

传统 SSD 的 FTL 通常以固定 page 或固定 block 为单位做逻辑到物理地址映射。但压缩后的数据是变长的：

- 有的数据压缩后可以放进一个 page 的剩余空间
- 有的数据压缩后可能跨越多个 page
- 有的数据不可压缩，可能直接以原始形式写入
- 逻辑页和物理页不再一一对应

DP-CSD 的 FTL 需要处理：

- 压缩后数据是否 fit in page
- page 是否已满
- 是否需要跨页写
- L2P mapping 更新
- metadata 更新
- GC 和 invalidation
- ECC 和掉电一致性

论文中 Figure 5 展示了 DP-CSD 的写入流程：host write 进入后先经过 CDPU compress，然后根据压缩后数据是否 fit in page 决定直接写 page buffer 还是进行 cross-page write，并更新 metadata 和 L2P mapping。

### transparent compression

DP-CSD 对主机呈现的是普通 NVMe SSD。也就是说，应用和文件系统可以不知道底层发生了压缩。

这有一个明显优点：

- plug and play
- 不需要修改应用
- 不需要修改文件系统
- 可以直接减少物理 NAND 写入和提升有效容量

但也有一个关键缺点：

- 上层应用无法基于压缩后的大小优化逻辑布局

例如 RocksDB 不知道 SSTable 在物理 NAND 上被压缩到了多小，因此它不会因为 DP-CSD 的透明压缩而减少 LSM-tree 层数或改变 compaction 策略。

这也是本文一个非常重要的系统启发：透明压缩可以优化底层物理资源，但不一定优化上层逻辑结构。

## evaluation setup

论文的评估平台是一台双路服务器，包含：

- Intel Xeon Platinum 8458P
- 88 CPU cores，2.7GHz
- 256GB DDR5 memory
- Intel QAT 8970 PCIe card
- Intel QAT 4xxx on-chip accelerators
- ScaleFlux CSD 2000 FPGA-based in-storage compression SSD
- DapuStor DP-CSD with DPZip
- Ubuntu 22.04.4

论文比较的对象包括：

- CPU software compression：Snappy、LZ4、Deflate、Zstd
- Peripheral CDPU：QAT 8970
- On-chip CDPU：QAT 4xxx
- In-storage FPGA CDPU：CSD 2000
- In-storage ASIC CDPU：DPZip / DP-CSD

评估指标包括：

- compression ratio
- throughput
- latency
- application throughput
- application latency
- CPU utilization
- power efficiency
- scalability
- multi-tenant isolation

应用层 workload 包括 RocksDB、Btrfs、ZFS 和 YCSB 等。

## compression ratio

论文首先比较不同算法在 Silesia dataset 上的 compression ratio。这里 compression ratio 定义为：

压缩后大小 / 原始大小

因此数值越低越好。

在 4KB 粒度下：

- Deflate / QAT 8970 大约 43.1%
- QAT 4xxx 大约 42.1%
- DPZip 大约 45%
- Snappy 和 LZ4 明显更差

DPZip 的压缩率略差于 Deflate / Zstd，是因为它的 LZ77 匹配为了硬件资源效率做了简化。但它明显优于轻量级压缩算法 Snappy 和 LZ4。

一个值得注意的点是：DPZip 固定以 4KB page 为压缩粒度，即使 host IO 是 64KB，DPZip 内部仍然按照 4KB page 压缩。因此在 64KB 场景下，QAT 的压缩率会进一步改善，而 DPZip 的压缩率基本保持稳定。

这说明 DPZip 的设计是 SSD page-oriented，而不是 large chunk-oriented。

## throughput

在 4KB block 上，论文报告的结果大致如下：

| 方案 | 压缩吞吐 | 解压吞吐 |
| --- | --- | --- |
| CPU Deflate | 4.9GB/s | 13.6GB/s |
| QAT 8970 | 5.1GB/s | 7.6GB/s |
| QAT 4xxx | 4.3GB/s | 7GB/s |
| DPZip | 5.6GB/s | 9.4GB/s |

一个反直觉结论是：

QAT 4xxx 虽然是更新的 CPU on-chip QAT，但在 4KB throughput 上并没有超过旧的 PCIe QAT 8970。

这说明 on-chip accelerator 的主要收益不是带宽，而是延迟。QAT 8970 有三个 co-processors，因此可以通过更高硬件并行度获得较好吞吐；QAT 4xxx 更靠近 CPU，但并不天然意味着更高 aggregate bandwidth。

在 64KB block 上，所有硬件 CDPU 的吞吐都会明显提升。原因是更大的 IO 粒度减少了 queueing、context switch 和 per-request overhead，也更容易利用 PCIe / DMA 带宽。

论文也指出，多个 DP-CSD 可以近似线性扩展。例如三个 DPZip-enabled computational storage drives 在 4KB 粒度下可以达到 16.3GB/s compression 和 20.9GB/s decompression。

## latency

延迟是这篇论文最核心的指标之一。

对于存储系统来说，即使吞吐很高，如果每个 4KB 请求的压缩延迟很大，也会严重影响数据库和文件系统的端到端性能。

论文报告：

- CPU Deflate 的 4KB 压缩延迟可达 70us
- QAT 8970 约为 28us compression / 14us decompression
- QAT 4xxx 约为 9us compression / 6us decompression
- DPZip 约为 4.7us compression / 2.6us decompression

这里体现了 placement 的影响：

### QAT 8970

QAT 8970 是 PCIe 外设。它需要：

- driver / library 生成 request descriptor
- QAT 通过 PCIe DMA 读取 descriptor
- QAT 通过 PCIe DMA 读取 source data
- QAT 执行压缩
- QAT 通过 PCIe DMA 写回 destination data
- 中断或 callback 通知完成

因此，即使 QAT 8970 本身计算能力不错，PCIe DMA 和 host-device 数据搬运仍然带来较高端到端延迟。

### QAT 4xxx

QAT 4xxx 集成在 CPU 侧，利用 CMI / DDIO / LLC 等机制缩短访问路径。论文提到 QAT 4xxx 的 64KB read latency 可以低到 448ns 级别，而 QAT 8970 的 PCIe DMA read latency 要高得多。

所以 QAT 4xxx 的优势主要体现在 latency，而不是 throughput。

### DPZip

DPZip 的延迟最低，因为它直接在 SSD write/read path 中压缩和解压。它不需要 host 和外设加速卡之间来回搬运数据。

本质上，DPZip 的优势来自减少数据移动，而不是单纯因为压缩引擎计算更快。

## data pattern sensitivity

压缩性能与数据可压缩性强相关。

对于可压缩数据，压缩器可以快速发现重复模式并输出较小结果；对于不可压缩数据，压缩器可能花很多时间尝试匹配，却发现没有收益。

论文发现：

- QAT 4xxx 在不可压缩数据上 compression throughput 下降约 67%
- QAT 4xxx 在不可压缩数据上 decompression throughput 下降约 77%
- DPZip 在不同可压缩性模式下吞吐波动控制在 15% 以内

这说明 DPZip 的 LZ77 设计较为保守，不会在不可压缩数据上花太多资源做无效匹配。它牺牲了一部分极限压缩率，但换来了更稳定的吞吐和延迟。

对于存储系统来说，这种稳定性很重要。因为真实 workload 中既有高度可压缩数据，也有已经压缩过的数据、加密数据或随机数据。系统需要关心 tail latency 和 performance predictability，而不是只看平均吞吐。

## application-level evaluation

论文强调：microbenchmark 结果不能直接代表应用性能。

### RocksDB

RocksDB 使用 LSM-tree。压缩方式会影响 SSTable 的大小，进而影响 LSM-tree 的层数、读放大和 compaction 行为。

如果压缩发生在 RocksDB 内部或文件系统层，RocksDB 可以感知压缩后的逻辑数据布局。这样 SSTable 会更小，可能减少 LSM-tree 深度和 read amplification。

但 DP-CSD 的 in-storage compression 对 RocksDB 是透明的。RocksDB 仍然以未压缩的逻辑大小组织 SSTable。因此，DP-CSD 虽然减少了底层物理写入和 NAND 占用，但不会自动改变 RocksDB 的逻辑结构。

这带来了一个重要结论：

应用可见压缩和透明底层压缩的系统效果不同。

- 应用可见压缩：可以改变上层数据结构
- 透明底层压缩：可以减少物理资源消耗，但不改变应用逻辑

### Btrfs / ZFS / QZFS

文件系统层压缩本身也有额外开销。论文指出，Btrfs、ZFS、QZFS 中的压缩涉及异步处理、额外内存拷贝、writeback pressure 和 metadata flushing 等问题。

因此，即使硬件压缩器本身很快，文件系统层的管理开销仍然可能掩盖真实压缩延迟，并降低端到端吞吐。

这再次说明，硬件加速器的效果必须放在完整软件栈中理解。

## power efficiency

论文区分了两个层次的功耗效率：

- CDPU module-level power efficiency
- full-system power efficiency

DPZip 作为单个压缩模块的能效非常高。论文提到 DPZip 消耗约 2.5W，而 CPU software compression 可能消耗约 132W，因此 standalone CDPU 的能效可以比软件压缩高很多。

但到了系统层面，能效提升不会完全等比例转化。因为系统中还有：

- CPU polling
- driver overhead
- memory copy
- PCIe data movement
- filesystem metadata
- application scheduling
- SSD internal management

因此，DPZip 的 standalone efficiency 很高，但 end-to-end system efficiency 的提升会被其他系统开销稀释。

不过总体来看，DP-CSD 仍然在 device、system 和 application 层面都表现出较高能效。论文报告 DPZip 在设备级 compression / decompression 上分别达到约 169.87MB/J 和 165.65MB/J，多设备扩展后还能进一步提升。

相比之下，QAT 的能效会受异步轮询、CPU busy waiting 和 CPU thread contention 影响，不一定明显优于软件压缩。

## scalability

DP-CSD 的一个重要优势是多设备扩展。

论文报告：

- 单个 QAT 4xxx accelerator 约 4.77GB/s
- 两个 QAT 4xxx accelerator 约 9.54GB/s
- 单个 DP-CSD 约 12.5GB/s
- 八个 DP-CSD 可扩展到约 98.6GB/s

QAT 4xxx 的扩展受 CPU socket 数量限制。QAT 8970 的扩展则受 PCIe slot、队列深度和外设部署复杂度限制。

DP-CSD 的扩展逻辑更自然：每增加一块 SSD，就增加一份压缩能力。对于存储系统来说，这是一种 storage-capacity-proportional compression capacity。

也就是说，DP-CSD 的压缩能力跟随 SSD 数量扩展，而不是集中依赖少数 CPU-attached 或 PCIe-attached accelerator。

## multi-tenant isolation

论文还评估了 SR-IOV 多租户场景。

实验方式是：

- 将设备划分为多个 Virtual Functions
- 每个 VF 分配给一个独立 VM
- 24 个 VM 同时运行 workload
- 观察每个 VM 的吞吐稳定性

结果显示：

- QAT 4xxx 和 QAT 8970 的 performance oscillation 很严重
- QAT-based solutions 的 coefficient of variation 超过 50%
- DP-CSD 的 CV 低于 0.5%

这说明一个很关键的问题：

支持 SR-IOV 不等于真正实现了内部资源隔离。

QAT 可以把设备虚拟成多个 VF，但多个 VF 仍然共享内部压缩资源、队列和调度机制。如果内部没有足够的 fairness / QoS 机制，就会产生 noisy neighbor 问题。

DP-CSD 的 VF 隔离效果更好，因为它本质上是 SSD front-end queue 和 IO scheduling 体系的一部分，能够更自然地结合 fair IO scheduling 和 queue resource management。

这对云环境非常重要。云厂商不仅需要平均吞吐高，还需要不同租户之间性能稳定、可预测。

## takeaways

### takeaway 1: placement determines data movement

CDPU 放置位置决定数据路径。

PCIe QAT 需要 host memory 和 accelerator 之间来回 DMA。QAT 4xxx 更靠近 CPU cache / memory，因此延迟较低。DPZip 直接在 SSD IO path 内部压缩，减少 host-CDPU 数据移动，因此延迟最低。

所以，硬件加速器不是孤立模块。它的系统价值很大程度上取决于数据需要走多远。

### takeaway 2: on-chip CDPU mainly improves latency, not necessarily throughput

论文一个很重要的发现是，QAT 4xxx 相比 QAT 8970 并没有在吞吐上全面领先。它的主要优势是低延迟。

这提醒我们，在评估硬件加速器时，不能只用“新一代”“片上集成”来判断性能。吞吐、延迟、扩展性和系统路径需要分开分析。

### takeaway 3: in-storage compression is not equivalent to application-visible compression

DP-CSD 的压缩对 host 是透明的。这使它非常容易部署，但也意味着应用不能直接利用压缩结果优化逻辑结构。

例如 RocksDB 中，应用层压缩可以减少 SSTable 逻辑大小；而 DP-CSD 只减少底层物理写入，不会自动减少 LSM-tree 层数。

因此，未来可能需要 application-aware in-storage compression，让应用、文件系统和 SSD 之间共享更多压缩信息。

### takeaway 4: hardware acceleration needs full-stack evaluation

硬件压缩器本身快，不代表端到端一定快。

真实系统中还有：

- driver overhead
- queueing overhead
- DMA overhead
- polling overhead
- memory copy
- filesystem metadata
- writeback
- application layout
- multi-tenant contention

这篇论文的价值在于，它不是只做 microbenchmark，而是把 CDPU 放在 RocksDB、Btrfs、ZFS、多设备、多 VM 和功耗场景中综合分析。

### takeaway 5: SR-IOV is not enough for accelerator isolation

QAT 的多 VM 实验说明，设备支持 VF 只是第一步。真正的性能隔离还需要内部资源调度、队列隔离和 QoS 机制。

这对所有硬件加速器都成立，包括 QAT、DSA、IAA、DPU、SmartNIC 和未来的 virtio accelerator。

## discussion

### CPU software compression

CPU 软件压缩的优势是灵活。

它可以随时切换算法：

- Snappy
- LZ4
- Deflate
- Zstd
- 不同 compression level
- 不同 chunk size

但是 CPU 压缩会消耗 core、cache 和 memory bandwidth。对于云数据中心来说，这会造成较高的机会成本。

### Peripheral CDPU

Peripheral CDPU 适合已有系统中的加速部署。它可以作为 PCIe 卡加入系统，不需要修改 SSD。

但是它的缺点是数据搬运路径较长。host memory 到 PCIe accelerator 的 DMA 往返会影响延迟。对于小块 IO 或 latency-sensitive workload，PCIe 外设型加速器不一定理想。

### On-chip CDPU

On-chip CDPU 减少了 PCIe latency，适合小块低延迟请求。但它的扩展性受 CPU socket 限制，且多租户场景下内部资源隔离仍然是问题。

### In-storage CDPU

In-storage CDPU 最适合减少数据移动和 NAND 写入。它的压缩能力随着 SSD 数量自然扩展，对云存储和多租户场景很有吸引力。

但它的主要问题是透明性带来的上层不可见。未来如果希望发挥更大应用性能收益，可能需要设计新的 host-device interface，让文件系统或数据库知道底层压缩信息。

## limitations

这篇论文也有一些局限。

### DPZip is not fully open

论文公开了 DPZip 的设计和评测，但它毕竟是厂商 ASIC。具体 RTL、完整微架构细节、精确面积功耗分解并未完全公开。因此，学术复现难度较高。

### algorithm comparison is not perfectly apples-to-apples

QAT 使用 Deflate，DPZip 是 Zstd variant，CPU 可以运行 Snappy、LZ4、Deflate、Zstd。不同算法之间本来就有不同压缩率和吞吐特征。

因此，这篇论文更像是真实产品形态的系统比较，而不是严格控制变量的算法比较。

### in-storage compression may introduce hidden FTL complexity

论文讨论了 DP-CSD 的 FTL 适配，但压缩带来的长期影响仍值得进一步研究。例如：

- compressed page fragmentation
- cross-page read amplification
- metadata consistency
- GC amplification
- capacity over-provisioning policy
- hot / cold data 的不同压缩策略
- incompressible data 的处理策略

### application-aware compression remains unsolved

DP-CSD 对应用透明，这有部署优势，但也限制了应用层优化。论文指出了这一问题，但没有完全解决应用与 SSD 协同的问题。

## possible future work

### application-aware in-storage compression

未来可以让 SSD 暴露压缩后大小、压缩率、page packing 状态等信息给文件系统或数据库，让上层应用基于真实物理布局做优化。

### compression-aware FTL

压缩后的数据是变长的，会给 FTL 带来新的映射、GC 和一致性问题。可以进一步设计 compression-aware FTL，针对 compressed segment 做更好的 page packing、wear leveling 和 read amplification 控制。

### unified CDPU virtualization layer

可以设计一个统一的虚拟化压缩设备框架，对 guest 暴露统一的 virtio-compress 或 DPDK compressdev 接口，而 host 后端可以根据硬件资源选择：

- CPU software compression
- QAT
- IAA
- DSA-assisted data movement
- in-storage compression SSD
- future DPU / SmartNIC compression engine

### QoS-aware accelerator scheduling

QAT 的 multi-tenant 实验证明，SR-IOV 不能自动保证性能隔离。因此，未来需要在压缩后端设计：

- per-VM queue isolation
- weighted fair scheduling
- request batching
- latency-sensitive priority
- noisy-neighbor detection
- hardware queue depth control

### co-design with databases and filesystems

压缩策略应该与 RocksDB、Btrfs、ZFS、LSM-tree、page cache、writeback 和 compaction 协同设计。否则底层硬件加速可能无法转化成上层应用收益。

## final thoughts

这篇论文的核心价值不只是提出了 DPZip，而是系统性地回答了“压缩加速器应该放在哪里”这个问题。

它告诉我们：

- PCIe 外设型 CDPU 受数据搬运延迟影响
- CPU 片上 CDPU 主要改善 latency，而不一定提升 aggregate throughput
- SSD 内嵌 CDPU 能减少 host-CDPU 数据移动，并随 SSD 数量扩展
- 透明 in-storage compression 不等于应用可见压缩
- 硬件加速器必须做 full-stack evaluation
- SR-IOV 不等于真正的性能隔离

压缩加速器是一个系统问题，而不是单纯的硬件模块问题。

在云计算和虚拟化环境中，压缩资源需要被抽象、调度、隔离和组合。无论底层是 QAT、DPZip，还是未来的 DPU / SmartNIC / CXL-attached accelerator，上层都需要一个合理的软件栈把它们组织起来。

这也正好对应到基于 DPDK 和 virtio 的压缩设备半虚拟化研究：它的意义不仅是让 guest 能够调用压缩硬件，更是为云环境中的压缩资源提供统一、高效、可隔离的系统抽象。