---
title: virtio-net
published: 2025-10-25
description: 'virtio-net驱动研究'
image: ''
tags: ["virtualization"]
category: 'notes'
draft: false 
lang: ''
---

# Virtio-net的简单发包流程
+ `start_xmit`
+ `xmit_skb`
+ `virtnet_add_outbuf`
+ 以上是设备相关代码，以下的都是virtio通用代码
+ `virtqueue_add_outbuf`
+ `virtqueue_add`
  + `virtqueue_add_packed`
  + `virtqueue_add_split`
+ `virtqueue_add_split`
+ `virtqueue_add_desc_split`
  + `sg`地址转换后，填充desc ring
    + `vring_map_one_sg` -> `virtqueue_add_desc_split`
  + 更新avail ring

## 在virtio-net中,desc中填充的地址，是哪里来的
在调用`virtnet_add_outbuf`前，会调用`skb_to_sgvec`函数，把skb指向的数据（虚拟地址）转成用一组scatterlist条目，每个条目都是struct page和页内偏移来描述对应的物理内存页。

在`virtqueue_add_split`中，通过`vring_map_one_sg`，调用`dma_map_page`函数，把每个sg条目映射为设备可用的DMA地址，将结果赋给addr变量。
