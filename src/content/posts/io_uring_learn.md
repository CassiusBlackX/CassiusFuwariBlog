---
title: io_uring_learn
published: 2025-12-12
description: 'io_uring相关特性的学习笔记'
image: ''
tags: ["linux", "async io"]
category: 'notes'
draft: false 
lang: ''
---

# An Introduction to the io_uring Asynchronous I/O Framework
主要来自[An Introduction to the io_uring Asynchronous I/O Framework](https://blogs.oracle.com/linux/an-introduction-to-the-io-uring-asynchronous-io-framework)这篇博客。

## introduction
io_uring是一个异步IO框架，在Linux kernel 5.1+引入的。提供了一个低延迟并且特性丰富的interface来提供给需要异步IO功能，但是倾向于由kernel来执行真正IO操作的应用。这和SPDK这样的框架拉开了显著的区别，SPDK这样是倾向于完全绕开内核，自己来做IO操作，因此他们也实现了自己的filesystem和其它一些特性。

## motivation
原生Linux AIO框架受到以下几个限制，而io_uring就是为了解决这些问题

io_uring的特性：
+ 不支持buffered I/O，只支持direct I/O
+ 在一些情况下，当遇到阻塞的时候，没有一个确定的行为
+ 每次I/O需要两次系统调用，一次用于提交请求，另一次用于回收结果
  + 每次提交需要复制64+8字节的数据，而每次回收需要32字节的复制

## communication channel
一个io_uring实例有两个ring,一个submission queue(SQ)和一个completion queue(CQ)，在内核和应用程序之间共享。queue都是单生产者单消费者，并且大小为2的幂

queue提供了一个无锁的访问interface,并自带memory barriers。

应用程序创建一个或多个SQ entries(SQE)，并且更新SQ的尾部。kernel消耗SQEs，并更新SQ的头部。

kernel创建CQ entries(CQE)用于一个或多个完成请求，并且更新CQ尾部。应用程序消耗CQEs并更新CQ的头部。

### 系统调用API
io_uring API有三个系统调用组成，分别是io_uring_setup, io_uring_register和io_uring_enter。

#### io_uring_setup
为一个异步IO创建上下文
```c
int io_uring_setup(u32 entries, struct io_uring_params *p);
```
io_uring_setup()系统调用创建至少带有`entires`各元素的SQ和CQ，并返回一个文件描述符可用于执行之后的针对io_uring实例的操作。SQ和CQ是在kernel与用户程序之间共享的，因此不存在数据拷贝的开销。

应用程序通过`params`来配置io_uring实例，并由kernel来返回配置成功的SQ和CQ的信息。

一个io_uring实例可以通过三种方式类配置
+ interrupt driven: 默认方式下，io_uring实例被配置为中断驱动的I/O。I/O会在调用io_uring_enter()的时候被提交，并可以通过直接查询CQ来回收结果
+ polled: 使用忙等待的方式来轮询I/O结果，而不是通过异步IRQ(Interrupt Request)来等候通知。要想使用这个方式，文件系统和块设备必须要能够支持轮询。忙等待提供了更低的轮询，但是会消耗更多的CPU资源。目前，这个特性只有当开启了O_DIRECT模式的时候能被使用。当一个read或者write操作被提交取给一个polled模式的上下文，应用程序必须自己通过io_uring_enter()来轮询CQ。对于一个io_uring，混合polled和non-polled模式的I//O是非法操作
+ kernel polled: 在这个模式下，一个kernel线程会被创建以执行对提交队列的轮询。一个这样方式创建的io_uring实例允许一个应用程序在不需要切换上下文到内核态的情况下提交I/O请求。通过使用SQ来提交新的SQE并监督CQ，应用程序可以在不调用任何一个系统调用的情况下提交和回收I/O请求。如果这个kernel线程在一个超出用户可配置的时间长度上都是idle，那么就会告知应用程序自己进入了idle状态后进入idle状态。当发生了这个情况之后，应用程序必须调用io_uring_enter来唤醒kernel线程。如果I/O保持忙，那么kernel thread永远不会sleep。

io_uring_setup()会在成功的时候返回一个新的文件描述符。应用程序可以把这个文件描述符提供给一个mmap系统调用来映射SQ和CQ，也可以传递给io_uring_register()或者io_uring_enter()系统调用

#### io_uring_register
把files或者用户缓冲区注册给一个异步I/O
```c
int io_uring_register(unsigned int fd, unsigned int opcode, void *arg, unsigned int nr_args)
```
io_uring_register()系统调用会把user buffer或者files注册给一个对应一个io_uring实例的fd。注册files或者user buffers允许kernel能够更长时间的持有kernel的内部结构体来处理files，或者让kernel能够创建更长时间的应用程序buffer的映射，从而降低每次IO的开销。

注册了的buffers会驻留在内存中，并计入用户的`RLIMIT_MEMLOCK`资源限制统计中。额外的，每个buffer最大只能有1GiB。目前来说，buffers必须得是匿名的，不基于文件的内存，如由malloc或者mmap+MAP_ANONYMOUS返回的内存。也支持巨页，但是巨页必须整个都被pin在kernel中（即不能被swap出），即使只有巨页的一部分被使用。

完全可以分配一个巨大的buffer，但是其实只使用其中的一部分来真正的做IO。

一个应用程序可以增加或者减少注册的buffer的大小，但是需要首先取消注册已经存在的buffers，然后再次调用io_uring_register来注册新的大小的buffer。

一个应用程序可以动态的更新注册了的files，而不需要首先去取消注册之前已经注册了的files

可以使用eventfd()来获取一个（IO）完成事件或者一个io_uring实例。如果希望这么做，那么可以通过这个系统调用来注册。

正在运行的应用程序的安全凭据可以注册到 io_uring，io_uring 会返回一个与该凭据关联的 ID。希望在不同用户或进程之间共享同一个io_uring实例的应用程序，可以在SQE（提交队列条目）的personality 字段中传入此凭据 ID。如果设置了该字段，对应的 SQE 将以这些凭据的身份执行操作。

#### io_uring_enter
初始化 和/或 完成一个异步I/O
```c
int io_uring_enter(unsigned int fd, unsigned int to_submit, unsigned int min_complete
                    unsigned int flags, sigset_t *sig)
```
io_uring_enter()使用之前已经创建了的共享的SQ和CQ来初始化或者完成一个I/O。一次系统调用可以同时提交新的I/O并返回已经完成的I/O。

参数中的`fd`是之前调用io_uring_setup时候返回的fd。`to_submit`指定要从SQ中提交的IO请求的数量。系统调用会尝试等待至少有`min_complete`个事件完成后再返回。如果io_uring实例被配置为poll模式，那么`min_complete`的意义会有一点不同。如果传如的值为0，那么kernel会返回一个已经完成的事件，不会阻塞。如果`min_complete`是一个非零值，那么kernel会在有任何事件完成的情况下就立刻返回。如果没有任何完成的，那么系统调用会一直轮询直到有一或多个完成，或者该进程超出了它的时间片。

需要注意的是，如果是中断驱动的I/O，那么一个进程可以在完全不切换到kernel的情况下检查以下CQ中是否有新的完成事件。

io_uring_enter支持非常多的操作，包括
+ open, close和stat files
+ read和write多个buffers或者pre-mapped buffers
+ Socket I/O操作
+ 同步文件状态
+ 异步的监测一系列的fd
+ 为ring中的某一个操作创建一个超时链接
+ 尝试取消正在执行的一个操作
+ 创建I/O链
  + 顺序执行链中的任务
  + 并行执行多条链

当系统调用返回的时候，表明一定数量的SQE已经被消费并提交，那么可以安全的重用SQE条目了。即使实际的 I/O 提交被推迟到异步上下文中执行（这意味着该 SQE 实际上可能尚未被提交），上述说法依然成立。如果内核后续仍需使用某个特定的 SQE 条目，它会自行保留该条目的私有副本。

## liburing
liburing提供一组更高级别的API，允许开发者不用和更低级的纯系统调用打交道，也有基础的功能。API也同时避免了重复编写诸如初始化io_uring实例等的代码。

例如，在从io_uring_queue_init()系统调用返回了一个fd之后，应用程序应当总是立刻去调用mmap()来把SQ和CQ映射，以供后续的访问。但是这一整个操作可以由一个liburing的函数实现
```c
int io_uring_queue_init(unsigned entries, struct io_uring *ring, unsigned flags);
```

### liburing的简单demo
```c
/* SPDX-License-Identifier: MIT */
/*
 * Simple app that demonstrates how to setup an io_uring interface,
 * submit and complete IO against it, and then tear it down.
 *
*/
#include <liburing/io_uring.h>
#define _GNU_SOURCE
#include <stdio.h>
#include <fcntl.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <liburing.h>

#define QD 4
#define BUF_SIZE 4096

int main(int argc, char **argv) {
  if (argc < 2) {
    printf("USAGE: %s: file\n", argv[0]);
    return 1;
  }

  struct io_uring ring;
  int ret = io_uring_queue_init(QD, &ring, 0);
  if (ret < 0) {
    fprintf(stderr, "queue_init: %s\n", strerror(-ret));
    return 1;
  }

  int fd = open(argv[1], O_RDONLY | O_DIRECT);
  if (fd < 0) {
    perror("open");
    return 1;
  }

  struct iovec *iovecs = calloc(QD, sizeof(struct iovec));
  void *buf;
  for (int i = 0; i < QD; i++) {
    if (posix_memalign(&buf, 4096, BUF_SIZE))
      return 1;
    iovecs[i].iov_base = buf;
    iovecs[i].iov_len = BUF_SIZE;
  }

  off_t offset = 0;
  for (int i = 0;;) {
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    if (!sqe) {
      break;
    }
    io_uring_prep_readv(sqe, fd, &iovecs[i], 1, offset);
    offset += iovecs[i].iov_len;
    i++;
  }

  ret = io_uring_submit(&ring);
  if (ret < 0) {
    fprintf(stderr, "io_uring_submit: %s\n", strerror(-ret));
    return 1;
  }

  int done = 0;
  int pending = ret;
  for (int i = 0; i < pending; i++) {
    struct io_uring_cqe *cqe;
    ret = io_uring_wait_cqe(&ring, &cqe);
    if (ret < 0) {
      fprintf(stderr, "io_uring_wait_cqe: %s\n", strerror(-ret));
      return 1;
    }

    done++;
    ret = 0;
    if (cqe->res != BUF_SIZE) {
      fprintf(stderr, "ret = %d, wanted %d\n", cqe->res, BUF_SIZE);
      ret = 1;
    }
    io_uring_cqe_seen(&ring, cqe);
    if (ret) break;
  }

  printf("Submitted=%d, completed=%d\n", pending, done);
  close(fd);
  io_uring_queue_exit(&ring);
  return 0;
}
```

### liburing copy
```c
/* SPDX-License-Identifier: MIT */
/*
 * Very basic proof-of-concept for doing a copy with linked SQEs. Needs a
 * bit of error handling and short read love.
 */
#include <stdio.h>
#include <fcntl.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <assert.h>
#include <errno.h>
#include <inttypes.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include "liburing.h"

#define QD  64
#define BS  (32*1024)

struct io_data {
    size_t offset;
    int index;
    struct iovec iov;
};

static int infd, outfd;
static unsigned inflight;


static int setup_context(unsigned entries, struct io_uring *ring)
{
    int ret;

    ret = io_uring_queue_init(entries, ring, 0);
    if (ret < 0) {
        fprintf(stderr, "queue_init: %s\n", strerror(-ret));
        return -1;
    }

    return 0;
}

static int get_file_size(int fd, off_t *size)
{
    struct stat st;

    if (fstat(fd, &st) < 0)
        return -1;
    if (S_ISREG(st.st_mode)) {
        *size = st.st_size;
        return 0;
    } else if (S_ISBLK(st.st_mode)) {
        unsigned long long bytes;

        if (ioctl(fd, BLKGETSIZE64, &bytes) != 0)
            return -1;

        *size = bytes;
        return 0;
    }

    return -1;
}

static void queue_rw_pair(struct io_uring *ring, off_t size, off_t offset)
{
    struct io_uring_sqe *sqe;
    struct io_data *data;
    void *ptr;

    ptr = malloc(size + sizeof(*data));
    data = ptr + size;
    data->index = 0;
    data->offset = offset;
    data->iov.iov_base = ptr;
    data->iov.iov_len = size;

    sqe = io_uring_get_sqe(ring);
    io_uring_prep_readv(sqe, infd, &data->iov, 1, offset);
    sqe->flags |= IOSQE_IO_LINK;
    io_uring_sqe_set_data(sqe, data);

    sqe = io_uring_get_sqe(ring);
    io_uring_prep_writev(sqe, outfd, &data->iov, 1, offset);
    io_uring_sqe_set_data(sqe, data);
}

static int handle_cqe(struct io_uring *ring, struct io_uring_cqe *cqe)
{
    struct io_data *data = io_uring_cqe_get_data(cqe);
    int ret = 0;

    data->index++;

    if (cqe->res < 0) {
        if (cqe->res == -ECANCELED) {
            queue_rw_pair(ring, BS, data->offset);
            inflight += 2;
        } else {
            printf("cqe error: %s\n", strerror(cqe->res));
            ret = 1;
        }
    }

    if (data->index == 2) {
        void *ptr = (void *) data - data->iov.iov_len;

        free(ptr);
    }
    io_uring_cqe_seen(ring, cqe);
    return ret;
}

static int copy_file(struct io_uring *ring, off_t insize)
{
    struct io_uring_cqe *cqe;
    size_t this_size;
    off_t offset;

    offset = 0;
    while (insize) {
        int has_inflight = inflight;
        int depth;

        while (insize && inflight < QD) {
            this_size = BS;
            if (this_size > insize)
                this_size = insize;
            queue_rw_pair(ring, this_size, offset);
            offset += this_size;
            insize -= this_size;
            inflight += 2;
        }

        if (has_inflight != inflight)
            io_uring_submit(ring);

        if (insize)
            depth = QD;
        else
            depth = 1;
        while (inflight >= depth) {
            int ret;

            ret = io_uring_wait_cqe(ring, &cqe);
            if (ret < 0) {
                printf("wait cqe: %s\n", strerror(ret));
                return 1;
            }
            if (handle_cqe(ring, cqe))
                return 1;
            inflight--;
        }
    }

    return 0;
}

int main(int argc, char *argv[])
{
    struct io_uring ring;
    off_t insize;
    int ret;

    if (argc < 3) {
        printf("%s: infile outfile\n", argv[0]);
        return 1;
    }

    infd = open(argv[1], O_RDONLY);
    if (infd < 0) {
        perror("open infile");
        return 1;
    }
    outfd = open(argv[2], O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (outfd < 0) {
        perror("open outfile");
        return 1;
    }

    if (setup_context(QD, &ring))
        return 1;
    if (get_file_size(infd, &insize))
        return 1;

    ret = copy_file(&ring, insize);

    close(infd);
    close(outfd);
    io_uring_queue_exit(&ring);
    return ret;
}
```