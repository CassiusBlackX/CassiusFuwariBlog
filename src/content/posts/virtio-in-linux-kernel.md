---
title: virtio-in-linux-kernel
published: 2025-10-11
description: linux内核中的virtio
image: ''
tags: ['virtualization', 'virtio', 'vhost']
category: 'notes'
draft: false 
lang: ''
---
# Linux内核中的virtio
## 什么是virtio
virtio是一个虚拟机在主机设备之上的抽象层。是一个接口，允许虚拟机通过最小化的，名为Virtio设备的东西访问宿主机上的设备。之所以说是“最小化”，是因为他们基本上就是只能够进行收发数据，而宿主机去做主要的初始化、维护和处理硬件。

让宿主机的硬件去做尽可能的工作，而让VirtIO只负责收发数据。这样把任务卸载，相比模拟一个设备的执行，效率会更高。

virtio的核心框架是有一个标准协议的。所有的virtio设备和他们的驱动abixu达到这个协议的要求，包括feature bits, status, configuration, general operation等。

## 为什么选择（不选择）Virtio
### virtualization和emulation的区别
enumlation中，软件来模拟一个硬件的所有行为。当宿主机上并没有或者不支持的设备，但是虚拟机中需要使用的时候，偏向于使用emulation。但是，如果通过emulation的方式，那么相当于把绝大部分的压力都放到了CPU上。

virtualization中，软件把宿主机的硬件切分出来一部分给虚拟机用，这里的“切分”一般是使用部分的硬件功能，让虚拟机以为它真的拥有这么一个硬件设备，但是实际上更类似于从宿主机上借用这个设备。需要明确的是，宿主机并不是被剥夺了使用硬件的权限，而是更类似于虚拟机共享。

### 什么时候选择或者不选择virtio
+ 选择emulation的时候
  + 在不适合的硬件上运行一个操作系统，如在PC上装MacOS
  + 在其它操作系统上运行一个不适配的软件，如在MacOS上运行Microsoft Word(Windows版)
  + 需要使用不支持的硬件
+ 使用virtualization的时候
  + 关注性能
  + 不需要特殊硬件
  + 多个虚拟机，高效的使用宿主机的硬件

## virtio架构
在qemu中，virtio前端驱动在guest的内核中，virtio后端驱动在hypervisor(qemu)中，前后端之间的通信是通过VirtQueues & VRings。具体的Notifications(VM-Exit, vCPU IRQ)被路由到KVM interruptions中。

### VirtIO Drivers(frontend)
在guest OS中，每一个VirtIO驱动都被当作一个内核模块。
+ virtio驱动的功能
  + 接受来自用户进程的IO请求
  + 把IO请求发送到对应的后端virtio设备
  + 从对应的地方获取执行结果

### VirtIO Devices(backend)
virtio设备存在于hypervisor中，具体来说，就是在qemu进程中。
+ virtio设备的功能
  + 接受来自前端驱动的IO请求
  + 处理请求，把请求卸载到对应的宿主机的硬件上
  + 把处理了的请求的数据提供给virtio驱动

### VirtQueues
virtqueue是一种数据结构，协助设备和驱动之间处理各种的vring操作。virtqueue共享在客户机的物理内存中，因此每一对virtio的驱动和对应设备都可以访问同一页的内存。

需要区别virtqueue和vring(virtio-ring)。其中，vring是virtqueue的主要组成部分，vring才是实际在virtio设备和驱动程序之间传递数据的数据结构。


以下是qemu中部分相关数据结构的部分内容
```c
// VirtQueue
VRing vring;
VirtQueueElement * used_elems;
uint16_t last_avail_idx;
bool last_avail_wrap_counter;
uint16_t shadow_avail_idx;
bool shadow_avail_wrap_counter;
uint16_t used_idx;
// ...
```
```c
// VRing
unsigned int num;
unsigned int num_default;
unsigned int align;
hwaddr desc;
hwaddr avail;
hwaddr used;
VRingMemoryRegineCaches *caches
```
```c
// VRingDesc
uint64_t addr;
uint32_t len;
uint16_t flags;
uint16_t next;
```
```c
// VRingAvail
uint16_t flags;
uint16_t idx;
uint16_t ring[]
```
```c
// VRingUsed
uint16_t flags;
uint16_t idx;
VRingUsedElem ring[];
```
```c
// VRingUsedElem
uint32_t id;
uint32_t len
```

除了`VRing`本身，`VirtQueue`还同时处理其它的一些flags,切片和回调函数（这些也都是关于如何处理VRing）

不过VirtQueue的具体组织方式是取决于guest OS，以及讨论的是用户空间(qemu)还是内核态的框架。此外，virtqueue的操作方式也取决于具体的virtio配置，如split和packed的virtqueues

以下是linux kernel中的virtqueue和vring的数据结构部分内容
```c
// vring_virtqueue
struct virtqueue vq;
bool packed_ring;
bool use_dma_api;
// ...
union { struct split {}, struct packed{}};
bool (*notify)(struct virtqueue* vq);
bool we_own_ring;
// ...
```
```c
// virtqueue
struct list_head list;
void (*callback)(struct virtqueue *vq);
const char* name;
struct virtio_device *vdev;
unsigned int index;
unsigned int num_free;
void * priv;
```
```c
// struct split
struct vring vring;
u16 avail_flags_shadow;
u16 avail_idx_shadow;
struct vring_desc_state_split* desc_state
dma_addr_t queue_dma_addr;
size_t queue_size_in_bytes;
// struct packed
struct  vring {};
bool avail_wrap_counter;
bool used_wrap_counter;
u16 avail_used_flags;
u16 next_avail_idx;
// ...
```
```c
// struct vring
unsigned int num;
vring_desc_t *desc;
vring_avail_t *avail;
vring_used_t *used;
// ...
```
```c
// vring_desc
__virtio64 addr;
__virtio32 len;
__virtio16 flags;
__virtio16 next;
```
```c
// vring_avail
__virtio16 flags;
__virtio16 idx;
__virtio16 ring[];
```
```c
// vring used
__virtio16 flags;
__virtio16 idx;
vring_used_elem_t ring[]
```
```c
// vring_used_elem
__virtio32 id;
__virtio32 len;
```
### vring
vring是virtqueue中的主要功能，也是核心数据结构，携带这真正要传输的数据。之所以被叫做"ring"是因为他们是一个环形数组。

+ 每个virtqueue一般都会有三个类型的vring
  + descriptor ring
  + available ring
  + used ring

#### descriptor ring
是一个`descriptor`的环形数组，其中descriptors是一个数据结构，描述一个data buffer
+ `addr`: guest的物理地址(GPA)
+ `len`: data buffer的长度
+ `flags`: 有`NEXT`,`WRITE`,`INDIRECT`等
  + `NEXT`: 在下一个描述符中还有更多的相关数据
  + `WRITE`: 这个buffer是不是只写的（设备可写的）
  + `INDIRECT`: 这个buffer包含这一个间接描述符表
+ `next`: desc.ring中的下一个描述符

当设置了`NEXT`标志位的时候，表示当前描述符的指向的数据缓冲区的数据的内容，将在`next`中的数据缓冲区中继续。当一个或多个描述符之间以这样的方式链接这的时候，通常称作*descriptor chaining*。并且对于descriptor chain来说，他们可以是由全部只读或者全部只写的描述符构成的

只有驱动程序可以向descriptor ring添加描述符，而设备仅当描述符的标志位表明该缓冲区可写，才能向设备可写的缓冲区中写入数据。

一个缓冲区只能是只读或者只写的，不可能同时具有两种属性。

在一个descriptor ring中，一个buffer的GPA和长度不能不能于其他的内存范围相互重叠。而在ring中，索引靠后的描述符指向的内存地址，并不一定就比前面的描述符指向的地址更高

#### available ring
是一个指向描述符的循环数组。即available ring中的每个条目都指向一个descriptor ring中的描述符（或描述符链的头）
+ `flags`: 参数标志位
+ `idx`: 下一个可用的avail ring条目的索引
+ `ring[]`: 实际的available ring数组

其中，`flags`字段表示avail ring的配置及其部分操作。`index`字段表示可用环中下一个可用的条目，驱动程序将在此放入对descriptor（或描述符链的头）的下一个引用。`ring`字段表示实际的avail ring数组，驱动程序将descriptor ring的引用存储在这里。

只有驱动程序才会配置并向available ring添加一条条目，而只有对应设备只能从中读取。

当在最开始的时候，在驱动程序向available ring添加第一条条目之前，available ring，没有任何的flag,也没有任何的条目，idx会被配置为`0`，因为对于这个available ring来说，下一个可用条目的索引就是0

#### used ring
类似于available ring，但是它是一个指向descripto ring中已经使用的descriptor的数组（即设备已经写入或读取了一个描述符的数据缓冲区）
+ `flags`: 参数标志位
+ `idx`: used ring中下一个可用的条目的索引
+ `ring[]`: 实际的used ring数组（是一个pair的结构体）
  + `id`: 当前这个元素在descriptor ring中的索引
  + `len`: 向descriptor的buffer中写入的数据的长度

与available ring正好相反，只有设备可以配置并且向used ring中添加条目，而对应的驱动程序可以读它


## VHost
不同于virtio的drivers & devices（其数据平面在Qemu进程中）不同，vhost可以将数据平面卸载到另一个宿主机的用户进程(Vhost-user)或宿主机内核中。这样做的动机是为了提升性能。因为在纯virtio的解决方案中，每一次驱动程序请求主机对其物理硬件进行处理时，都会发生一次上下文切换，这些上下文切换是开销较大的操作，会在请求之间引入显著的延迟。通过将数据平面卸载到另一个主机用户进程或内核中，就可以绕过qemu进程，减少延迟，提升性能。

+ 因此和virtio对比，vhost的
  + 数据传输层(data plane数据面)现在是直接从guest kernel到host kernel了
  + virtio设备模型仍然存在，但其功能被缩减为仅处理控制平面任务

## VirtIO in Qemu
通过分析virtio-SCSI来简单理解一个virtio device是如何工作的，并探究virtqueue和vring在其中扮演的作用。

在下面的例子中，一个virtio-SCSI,使用split的virtqueue配置，并且协商使用`VIRTIO_VRING_F_EVENT_IDX`

### Virtio-SCSI
virtio-SCSI设备用于将虚拟逻辑单元（如硬盘驱动器）进行分组，并通过SCSI协议实现与它们的通信。在市里中，假设仅使用该身背连接一个硬盘驱动器。此设备（使用硬盘）的qemu启动参数可能包含如下内容
```sh
-device virtio-scsi-pci
-device scsi-hd,drive=hd0,bootindex=0
-drive file=/home/qemu-imgs/test.img,if=none,id=hd0
```
在qemu源码中，`hw/scsi/virtio-scsi.c`，可以看到相关函数，和该设备的操作相关。下面来看看设备是如何配置的，尤其是它的virtqueue

在qemu中，"realize"一词用于表示virtio设备的初始设置和配置（"unrealize"取消实现，用于拆除设备）
```c
// hw/scsi/virtio-scsi.c
void virtio_scsi_common_realize(DeviceState *dev, VirtIOHandleOutput ctrl, VirtIOHandleOutput evt, VirtIOHandleOutput cmd, Error ** errp) {
    // ...
    s->ctrl_vq = virtio_add_queue(vdev, s->conf.virtqueue_size, ctrl);
    s->event_vq = virtio_add_queue(vdev, s->conf.virtqueue_size, evt);
    for (i = 0; i < s->conf.num_queues; i++) {
        s->cmd_vqs[i] = virtio_add_queue(vdev, s->conf.virtqueue_size, cmd);
    }
}
```
绝大多数virtio设备都会有多个virtqueue，每个virtqueue都有他们自己独特的功能。在virtio-SCSI的情况下，我们有一个控制队列virtqueue(`ctrl_vq`),一个事件virtqueue(`event_vq`)和一个或多个命令/请求virtqueue(`cmd_vqs`)

控制virtqueue是用来做任务管理功能的，如启动，停止，重置SCSI设备。它同时也用来做定远和查询异步通知

事件virtqueue是用来报告主机上连接到virtio-SCSI的逻辑单元的相关信息（事件）。这些事件包括传输事件（例如设备重置、重新扫描、热插拔、热卸载等）、异步通知以及逻辑单元号(LUN)参数的更改

命令virtqueue用于普通的SCSI命令传输（如读写文件）。在接下来中，将关注命令virtqueue。

#### 命令virtqueue
如前文提到的，qemu中的virqueue结构体有一个用来处理输出的回调函数字段，叫做`VirtIOHandleOutput handle_output`。在virtio-SCSI的命令virtqueue中，这个回调函数字段会指向一个`virtio_scsi_handle_cmd()`函数
```c
// hw/scsi/virtio-scsi.c
static void virtio_scsi_device_realize(DeviceState *dev, Error **errp) {
    VirtIODevice *vdev = VIRTIO_DEVICE(dev);
    VirtIOSCSI *s = VIRTIO_SCSI(dev);
    Error *err = NULL;

    virtio_scsi_common_realize(dev, virtio_scsi_handle_ctrl, virtio_scsi_handle_event, virtio_scsi_handle_cmd, &err);
    // ...
}
```
```c
// hw/virtio/virtio.c
VirtQueue *virtio_add_queue(VirtIODevice *vdev, int queue_size, VirtIOHandleOutput handle_output) {
    // ...
    vdev->vq[i].vring.num = queue_size;
    vdev->vq[i].vring.num_default = queue_size;
    vdev->vq[i].vring.align = VIRTIO_PCI_VRING_ALIGN;
    vdev->vq[i].handle_output = handle_output;
    vdev->vq[i].used_elems = g_malloc0(sizeof(VirtQueueElemen) * queue_size);
    return &vdev->vq[i];
}
```
virtqueue的输出处理函数的调用方式取决于具体的virtio设备以及该virtqueue在设备中的角色。在virtio-SCSI的命令virtqueue的情况下，当一个virtio-SCSI驱动程序向qemu发送一个通知，告诉qemu去通知对应的设备在available vring上有一个SCSI命令数据准备就绪需要处理的时候，该virtqueue的输出处理函数就会被调用。

+ 一个virtio设备直到对其对应的virtio驱动程序完成以下操作后，才会参与到vring的操作中
  1. 向descriptor ring中添加了新的描述符
  2. 把对这个新的描述符的引用添加到了available ring中
  3. 通知设备available ring已经准备就绪，等待处理 

换句话说，当执行到了`virtio_scsi_handle_cmd()`函数的时候，意味着virtio-SCSI设备已经从它的驱动程序处收到了通知，并且开始准备处理它的命令virtqueue的available ring中的数据了。

`virtio_scsi_handle_cmd()`函数就基本上是一个对于`virtio_scsi_handle_cmd_vq()`函数的封装
```c
// hw/scsi/virtio-scsi.c
bool virtio_scsi_handle_cmd_vq(VirtIOSCSI *s, VirtQueue *vq) {
    VirtIOSCSIReq *req, *next;
    int ret = 0;
    bool suppress_notifications = virtio_queue_get_notification(vq);
    bool progress = false;

    QTAILQ_HEAD(, VirtIOSCSIReq) reqs = QTAILQ_HEAD_INITIALIZER(reqs);

    do {
        if (suppress_notifications) {
            virtio_queue_set_notification(vq, 0);
        }
        while ((req = virtio_scsi_pop_req(s, vq))) {
            progress = true;
            ret = virtio_scsi_handle_cmd_req_prepare(s, req);
            if (!ret) {
                QTAILQ_INSERT_TAIL(&reqs, req, next);
            } else if (ret == -EINVAL) {
                /* th device is broken and shouldn't process any request */
                while (!QTAILQ_EMPTY(&reqs)) {
                    // ....
                }
            }
        }
        if (supress_notifications) {
            virtio_queue_set_notification(vq, 1);
        }
    } while (ret != -EINVAL && !virtio_queue_empty(vq));

    QTAILQ_FOREACH_SAFE(req, &reqs, next, next) {
        virtio_scsi_handle_cmd_req_submit(s, req);
    }
    return progress
}
```
这个函数告诉了我们virtio-SCSI的命令virtqueue将如何处理它的available ring上的数据

在接下来的场景中，首先回顾一下，已经协商了`VIRTIO_VRING_F_EVENT_IDX`功能位。对于设备而言，这意味着只有当*used ring*中的`idx`等于*available ring*中的`idx`的时候，才会通知驱动程序。

换句话来说，如果驱动程序在available ring上添加了20条条目，索引从0到19，则available ring的`idx`在添加完最后一条条目(`ring[19]`)之后将变为`20`。在设备处理完了available ring中的最后一条条目(`ring[19]`)并且把它放到了对应的used ring中后，used ring的`idx`也将变为`20`。这时候，几乎立刻在添加完最后一条条目到used ring中后，设备要立刻通知它对应的驱动程序。

回到`virtio_scsi_handle_cmd_vq()`。

在函数开始的时候，一个virtio-SCSI请求(`VirtIOSCSIReq`)的队列`reqs`，被初始化了
```c
QTAILQ_HEAD(, VirtIOSCSIReq) reqs = QTAILQ_HEAD_INITIALIZER(reqs);
```
对于virtio-SCSI的命令virtqueue，它的available ring上的每一个条目都被转变成了一个`VirtIOSCSIReq`对象并被添加到`reqs`队列的末尾。

在开始读取avaialble ring之前，首先要抑制设备向驱动程序发送的通知，毕竟设置了`VIRTIO_VRING_F_EVENT_IDX`。接下来，进入到第二个while循环`\while ((req = virtio_scsi_pop_req(s, vq)))`。在这个while循环里面，我们遍历available ring，并对每一个条目，都把它转变为一个`VirtIOSCSIReq`对象并添加到了`reqs`队列的尾部

在读取完了能够从available ring中读取的所有内容后，便会重新启用通知，以便设备在处理完请求并将其放入used ring中后，能够通知驱动程序有已经处理了的数据。需要注意的是，这里仅仅是启用了通知，并没有发送通知。在我们的场景下(`VIRTIO_VRING_F_EVENT_IDX`)，这仅仅是做好了准备，一旦我们将所有已处理请求的数据都放入used ring后，就可以通知设备

在提交请求之前，可以看到我们仍然在do-while循环中，只有当设备出问题或者我们没有在available ring中留有未读数据的时候才会退出循环。这是为了防止在我们读完了available ring中的最后一条条目后又有新的数据添加了进来

现在设备已经读完了它在available ring中可以督导的所有数据并把每一条条目都添加到了它自己的`VirtIOSCSIReq`对象中，我们接下来就遍历`reqs`并一条一条的把请求提交给真正的SCSI硬件设备去处理。

一旦一个请求被host上的SCSI设备完成，将会去执行`virtio_scsi_command_complete()`，然后是`virtio_scsi_complet_cmd_req()`，最后到`virtio_scsi_complete_req()`。其中比较有趣的函数是`virtio_scsi_complete_req()`，因为这个函数是设备把使用了的数据放入used ring中的地方。

```c
// hw/scsi/virtio-scsi.c
static void virtio_scsi_complete_req(VirtIOSCSIReq *req) {
    VirtIOSCSI *s = req->dev;
    VirtQueue *vq = req->vq;
    VirtIODevice *vdev = VIRTIO_DEVICE(s);

    qemu_iovec_from_buf(&req->resp_iov, 0, &req->resp, req->resp_size);
    /* Push used request data onto used ring */
    virtqueue_push(vq, &req->elem, req->qsgl.size + req->resp_iov.size);
    /* Determin if we need to notify the driver */
    if (s->dataplane_started && !s->dataplane_fenced) {
        virtio_notify_irqfd(vdev, vq);
    } else {
        virtio_notify(vdev, vq);
    }

    if (req->sreq) {
        req->sreq->hba_private = NULL;
        scsi_req_unref(req->sreq);
    }
    virtio_scsi_free_req(req);
}
```

为了完成一个请求，virtio-SCSI设备必须通过把处理了的数据放到used_ring上(`virtqueue_push()`)来使驱动程序能够访问这些数据。需要注意的是，真正的数据现在已经被写入了描述符的buffer中。我们现在做的是告诉驱动程序去去descriptor ring的哪里去寻找，以及（如果有）我们向它的data buffer中写了多少数据。

# vhost
[原文链接](https://blogs.oracle.com/linux/post/introduction-to-virtio-part-2-vhost)
## overview
虽然经典的VirtIO配置对于通用工作负载已经能够提供良好的性能，但在处理更高强度的工作负载时，其数据平面的局限性便暴露了出来，更具体的说，是Qemu参与数据平面所带来的开销问题。

vhost是一个对VirtIO的优化，用于把对数据平面的处理从qemu卸载到其他更有效率的组件上。

首先，我们将分析为什么纯VirtIO配置会带来开销。我们将要探讨其中涉及的高成本的操作，如VM-Exits, VM Entries,上下文切换以及Qemu在用户空间的处理。接下来我们将介绍vhost并详细说明它如何通过把数据平面移动到内核（基于内核的vhost）或者一个专用的用户空间中，来解决这些问题。

然后我们会同时深入讨论基于内核的vhost和vhost-user的架构，包括他们的优势、劣势、典型使用场景，以及他们如何会影响性能。

为了能够更形象的说明开销的主要来源，将分别分析纯VirtIO、基于内核的vhost和vhost-user对于网络的配置。我们将逐步分析他们的发送和接受的执行路径，精确指出在每种情况下开销发生的位置和时机。

精准初步的，我们将总结每种配置的执行路径，并对Tx和Rx路径的开销做粗略估算的方程。这些开销方程是为了展示和对比不同配置的主要开销来源。

## Context: VirtIO Data Plane Overhead
在介绍vhost之前，我们需要首先理解vhost是为了解决什么问题: 在VirtIO数据平面中的开销

### Limitations of Pure VirtIO
实现vhost的主要动力就是为了减少在高负载场景下在VirtIO的数据平面处理数据的开销。对于一个使用纯VirtIO配置的虚拟机，如果每个任务涉及到的IO负载都比较低，比如浏览网页或者播放视频，那么基本不会遇到什么可以感知到的开销。但是如果吞吐量飙升，开销就会逐渐变得可感知，最终成为系统的瓶颈，导致性能下降

在VirtIO中，guest和host之间通过virtqueue交换数据。但是在一个纯VirtIO环境中，对virtqueue的读写、处理终端、转发数据包（或者IO请求）都是由qemu在用户空间处理的。换句话来说，qemu是作为一个位于guest和host内核之间的中间人，而导致了在数据平面的操作中的开销。在一个纯VirtIO配置中的主要问题就是**上下文切换**和**VM Exit/entry**，以及qemu的设备模拟

假设在一个纯VirtIO配置的场景中，运行着一个*virtio-net*设备。qemu正在用户空间中模拟这个设备，意味着数据包将需要在guest的内核、qemu和host内核之间移动。

现在，假设我们没有使用任何其他的优化方式，如`ioeventfd`,`irqfd`,`VIRTIO_F_EVENT_IDX`或`NAPI`.

+ 在以上的条件下，当guest想要发送一个包，需要经历**1个上下文切换**，**1个VM-Exit**和**1个VM-Entry**
  + Guest => Host Kernel: VM-Exit到KVM中
  + Host kernel => Qemu: 上下文切换，从kernel到用户空间(qemu)
  + Host kernel => Guest: 通过VM-Entry回到Guest中
+ 当要接收一个包，也涉及到**1个上下文切换**，**1个VM-Exit**和**1个VM-Entry**
  + Host kernel => Qemu: 上下文切换，从内核到用户空间
  + Guest => Host kernel: VM-Exit到KVM
  + Host kernel => Guest: 最终VM-Entry回到guest

##### Guest=>Host transmit path
+ `virtio-net`驱动程序把数据放入`virtio-net` Tx VirtQueue中
+ `virtio-net`驱动程序向doorbel register执行写操作，告诉host(KVM)去唤醒`virtio-net`设备
  + **VM-Exit(Guest->Host Kernel)**: 向doorbell register执行写操作导致CPU退出guest模式，陷入KVM并进入到host kernel中
+ KVM拦截消息并知道了`virtio-net`设备(qemu中)需要被唤醒
+ KVM转发这这一唤醒消息给qemu这样`virtio-net`设备就可以处理新的Tx virtqueue中的数据
  + **上下文切换(Host kernel->userspace)**: 唤醒Qemu意味着CPU需要切换上下文到Qemu进程(在用户态中)
+ `virtio-net`设备从`virtio-net` Tx virtqueue中读取数据
+ Qemu通过一个系统调用，把数据复制到host内核中的网络栈中
  + 这是一种内核态与用户态之间的环级切换，而非上下文切换——CPU现在仍然在Qemu的执行上下文中，只是在执行内核代码
+ TAP/物理网卡把这个包发送到网络中
  + **VM-Entry(Host->Guest)**: host执行结束后就返回到Guest中

##### Host=>Guest receive path
+ TAP/物理网卡从网络中收到了数据
+ TAP/物理网卡把数据追加到host内核网络协议栈并通过一个文件描述符通知qemu
  + **上下文切换(Host Kernel=>Userspace)**: 当数据准备好了，内核唤醒Qemu并且CPU上下文切换到Qemu所在的用户态进程
+ Qemu使用一个系统调用(e.g.,`read()`/`recvmsg()`)来把数据从TAP设备中复制出来
+ Qemu把数据写入到`virtio-net`的Rx virtqueue中
+ Qemu发出一个系统调用(`ioctl()`)给KVM，请求给Guest注入一个中断
  + 这是一个环级切换，不是一个上下文切换——CPU现在仍然Qemu的执行上下文中，只是在执行内核的代码（为了处理中断）
+ KVM手动给guest注入一个中断，通知其`virtio-net`的Rx virtqueue中有新的数据
  + **VM-Exit(Guest=>Host Kernel)**: 如果客户机保持活跃（一般来说都是的），那么CPU需要短暂退出客户机模式，注入中断
  + **VM-Entry(Host Kernel=>Guest)**: 在给guest注入完中断后，KVM导致CPU回到Guest模式（恢复到guest的vCPU的上下文）
+ `virtio-net`驱动程序接收到中断并被告知有新的数据在`virtio-net`的Rx virtqueue中
+ `virtio-net`驱动程序从Rx virtqueue中读取数据并把它递交给内核的网络协议栈中

#### 开销
为了突出纯 VirtIO 架构（特别是使用 virtio-net 进行网络通信时）的开销成本，我们将为单个数据包的接收和发送过程建立通用的、近似的开销计算公式。当然，上下文切换和虚拟机退出/进入并不是此处唯一的开销来源。其他开销还包括宿主机内核网络协议栈的处理、Qemu 的处理以及数据拷贝等。然而，需要注意的是，上下文切换和虚拟机退出/进入是主要的开销来源，因此它们的开销权重相比其他开销成本要更高。

+ Tx路径的主要开销在
  + 在KVM中处理来自客户机的kick操作所引发的VM-Exit
  + 当qemu被唤醒去处理来自guest的新的Tx数据（如果之前处于休眠状态的话）所引发的上下文切换
  + qemu的处理，包括设备模拟逻辑、系统调用和其他的相关任务
  + qemu把数据从Tx virtqueue中拷贝到可被TAP设备访问到的共享内存中
  + host的内核网络协议栈转发数据到物理NIC上进行处理
  + 最终回到guest的VM-Entry
+ Rx路径的主要开销
  + host内核网络协议栈的处理开销，包括从NIC获取数据并把他们准备好以备Qemu取走
  + qemu被唤醒以处理来自TAP设备或物理NIC的新的Rx数据
  + qemu把数据从共享内存复制到Rx virtqueue时候的内存拷贝
  + qemu的处理，包括设备模拟、系统调用和其他的相关任务
  + （如果此前虚拟机正在运行）为了注入中断而导致的VM-Exit
  + guest为了处理中断并处理Rx virtqueue中的数据的VM-Entry

## 什么是vhost
vhost是一个协议，允许virtio数据平面被卸载到其他组件上——如一个专用的内核驱动或一个独立的用户空间进程——来提高性能。+

为了实现这个方式，一般是通过我们正在使用的**kernel-based vhost**(e.g., `vhost-scsi`, `vhost-net`)或者一个独立的用户空间进程(e.g., DPDK的`vhost-user`)。这两种方式都在主要的数据通路中代替了qemu，不过前者是移动进了内核，后者是移动到了另一个用户空间进程。一般来说，kernel-based vhost使用更见常见一点

### kernel-based vhost
kernel-based vhost通过把数据平面的virtqueue的处理直接移动进入内核中来提高virtio的性能。更具体的来说，virtqueue的处理被从qemu(e.g.e, `virtio-net`设备模拟)到了一个专用的内核驱动(e.g.,`vhost-net`)中。每一个vhost驱动都可以生成一个或多个内核线程来处理virtqueue。

在原来的方式中，执行的过程必须经历以下几个阶段
```
guest => virtio driver => host kernel => qemu(userspace) => host kernel => TAP device/Physical NIC
```
而现在把数据路径移入内核并且加上若干优化，可以变成下面这个样子
```
guest => virtio driver => host kernel(vhost-net) => TAP device/Physical NIC
```
这样做了之后，就消除了在host内核和Qemu(用户空间)之间进行上下文切换的需求，降低了CPU开销和延迟。在这之上，因为数据路径在内核中，我们可以利用内核级别的优化技术，如网络卸载、批处理、零拷贝技术，来获得更好的网络或存储IO性能

+ 现在已有的一些kernel-based的vhost驱动有
  + `vhost-net`
  + `vhost-scsi`
  + `vhost-blk`
  + `vhost-vsock`
  + `vhost-vdpa`: 为任何使用vDPA(VirtIO Data Path Acceleration)框架的Virtio设备的硬件加速功能，通过一个vhost兼容的接口，允许直接的从guest到硬件的访问

其中，`vhost-vdpa`驱动和普通的vhost使用场景有一点不同。不同于其他的vhost驱动——他们都直接实现了kernel-based的数据路径——`vhost-vdpa`是作为一个框架，把任何兼容的virtio设备直接卸载到硬件的数据路径。


### userspace-based vhost(vhost-user)
`vhost-user`通过把virtqueue的处理卸载到了一个独立的用户空间进程（如DPDK或SPDK）中来有效的优化了VirtIO的数据路径效率。更准确的来说，`vhost-user`是一个协议，让这些独立的用户空间进程能够像一个VirtIO设备一样执行virtio后端的功能。

不同于kernel-based vhost, `vhost-user`并没有一个专门的内核驱动。实际上，`vhost-user`本身甚至都不是一个进程或线程。相反，它是一个Qemu内部的更能，告诉qemu把这个VirtIO设备对应的数据平面卸载到一个单独的用户空间进程去。

但是如果数据平面如果仍然在用户空间的话，性能不还会像纯VirtIO配置一样，受到上下文切换带来的影响吗？ 答案是是的，但是`vhost-user`可以联同其它guest侧的优化技巧，如NUMA-pinning和轮询技术。使用这些优化技巧，可以有效的减少甚至消除VM-Exit/Entry，中断和上下文切换带来的开销

需要注意的是，上面提到的这些guest侧的优化技巧和其它的host侧的优化技巧都是完全独立的，即并不是针对`vhost-user`场景下的。

传统上来说，网络和存储设备的错做都是使用一种**事件驱动**模型，其中的线程只有在新的数据到达的时候才会被唤醒。但是在高性能场景下，这个事件驱动的方式，因为要频繁处理中断和唤醒线程的开销，而会带来很多的延迟

相比之下，轮询，经常被和`vhost-user`一起来使用。有了轮询，我们需要一个专门的CPU核心来不断的检查是否有新的数据。一般来说，在低吞吐的负载下，轮询是非常资源浪费的，毕竟CPU花费了更多的时间在检查是否有新数据，而非真正的处理数据

但是，当工作负载长期很高的时候，轮询会变得很有用。因为，持续的轮询是否有新的数据，消除了频繁的唤醒线程带来的开销——线程一直都活跃。就算负载不是上时间的高，NUMA-pinning技术和轮询在低延迟要求的场景下也是非常有用的，因为它避免了中断和上下文切换带来的额外抖动或延迟。

甚至还有更进一步的优化方式来让`vhost-user`有更高的性能，如在用户空间进程直接通过VFIO来直接管理网卡，从而绕过内核的网络协议栈，消除原本在收发数据包时候产生的内核网络协议栈处理的开销。

## vhost架构
vhost的架构仍然是包含了三个关键的部分，前端，后端和virtqueue&vring

在一个纯VirtIO配置中，前端驱动程序在guest的那个中，后端设备在Qemu(userspace)中，而他们之间的消息传递依赖于virtqueu & vring，在共享内存中。

当切换到一个vhost配置中，最大的变化就是后端。对于一个kernel-based vhost，后端被移动到了host的内核中，并有一个专门的内核驱动。对于一个`vhost-user`，后端移动到了另外一个host中的用户空间进程。

在虚拟化中，一个**设备模型**代表一个对于guest来说的虚拟的设备接口，但是并不真正的实现完整的功能。有了vhost之后，数据平面被移出了qemu，但是控制平面的部分，仍然在驱动程序和qemu中的virtio设备之间存在。既然现在qemu只是维护了设备的配置和状态，它不再处理数据，因此只是一个设备模型，而不是一个真正的后端。

### kernel-based vhost架构
在接下来的场景中，将假设采用了以下几种优化
+ `ioeventfd`: 当有新的Tx数据，要通知后端的时候，避免额外的在内核和用户空间之间的上下文切换
  + 如果没有这个，就需要上下文切换到qemu并处理消息通知，然后切换回内核来开始处理新来的TX数据
  + 有了这个，`vhost-net`就昆虫直接处理消息通信然后立刻开始处理数据了
+ `irqfd`: 当有新的Rx数据的时候，避免额外的内核和用户空间(qemu)之间的上下文切换
  + 没有这个，需要切换到qemu来处理通知然后做一个环级转换回kernel来把中断注入到guest中
  + 有了这个，我们可以在内核内把中断标记为“待处理”状态，稍后再通知guest

#### overview
现在qemu中的virtio-net设备让开了，直接允许virtio-net的数据平面的virtqueue在驱动程序和`vhost-net`之间共享。因为qemu中的`virtio-net`设备不在处理数据，因此它只是一个设备模型。它的唯一的功能只和控制平面的操作相关

同样值得注意的是，对于控制平面的操作，qemu中的`virtio-net`设备模型使用`ioctl()`系统调用来和`vhost-net`内核模块通信(e.g.,`ioctl(fd, VHOST_SET_MEM_TABLE, mem)`)。这些调用是用于通知vhost-net一些如下的事情
+ guest的内存区域: 虚拟机的哪些内存区域存放了virtqueue
+ virtqueue的配置信息: 每个数据平面的virtqueue的地址，大小，eventfd(用来中断和kick)
+ feature/offload配置信息: 与guest协商的是否启用或禁用特定的卸载功能(如TSO、校验的卸载)
+ 运行时变更与销毁: 更新vring地址，开始或停止virtqueue，销毁时候释放资源，热迁移管理等

