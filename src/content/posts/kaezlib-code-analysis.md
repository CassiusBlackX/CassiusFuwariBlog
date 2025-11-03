---
title: kaezlib-code-analysis
published: 2025-10-02
description: 'analyse how kaezlib dump workload to accelerator'
image: ''
tags: ["kae"]
category: notes
draft: false 
lang: ''
---
# kaezlib源码简单分析
## 摘要
+ kaezlib仓库自带zlib，并通过patch的方式，修改zlib源码，把原有API（如`deflateInit2_`,`deflate`,`deflateEnd`;`inflateInit2`,`inflate`,`inflateEnd`等）包裹/插桩，在调用原始实现（带`lz_`前缀）前。先调用`kz_`前缀的适配层，由其决定是否把这次操作发给KAE加速器或回退到原生软件实现
+ 数据交换是零拷贝/直指针的方式（把`z_stream`里的`next_in/next_out`直接交给驱动），但有分块chunk管理、头尾格式(zlib/gzip header/tail)处理、以及当输出空间不够时的缓冲机制

## 实现
### zlib源码的修改
对于zlib的修改，只发生在`deflate.c/h`,`inflate.c/h`上

+ 将原有的实现改名或分离出“原生zlib”实现前缀为`lz_`，保留旧的名字作为接入点
+ 新增带`kz_`前缀的函数集（声明在`kaezip.h`），以及在原始API层增加一个决策/路由层，e.g.
  + `deflateInit2_`新实现会
    + 在编译时（若定义`CONF_KAEZIP`）检测`kz_get_devices()`(是否存在KAE设备)
    + 若有设备：先初始化lz(软件)状态，再调用`kz_deflateInit2_`(适配层)来完成KAE相关初始化；或直接回退到软件
  + `deflate`(实际压缩入口)被实现为：在`CONF_KAEZIP`情况下先询问`kz_get_devices()`，若有设备则调用`kz_deflate`，并将`kz_deflate`的返回码用于决定是否调用`lz_deflate`(软件路径)。`kz_deflate`在无法使用硬件时会返回`Z_CALL_SOFT`，上层再软降级调用`lz_deflate`
+ 在`deflate_state`结构中加入`kaezip_ctx`字段，用来保存KAE/kaezip的上下文指针；也有其它字段(header_pos等)用于格式头尾管理
+ 修改`compressBound`/`deflateBoud`的估算逻辑以配合KAE行为
+ 修改Makefile.in,增加`KAEZIP_CFLAGS`和`KAEZIP_LDFLAGS`，把`libkaezip.so`,`libwd`,`libwd_comp`等加入链接路径并定义`-DCONF_KAEZIP`

以上的修改就把zlib的API和实现层分离，API层负责路由，实现层有`kaezip`库提供具体对接

### 设备探测
+ `kz_get_devices()` (`kaezip_common.c`):
  + 实现：打开`/sys/class/uacce`目录，查找名字包含`hisi_zip`的设备节点
  + 返回1表示能够找到`wrapdrive/hisi_zip`(v1风格)，否则返回0
  + 这是最早的简单监测(用于在`deflateInit2_`这种入口判断是否要启用KAE逻辑),即是否存在KAE设备
+ `uadk_get_accel_platform()` (`kaezip_adapter.c`):
  + 进一步检查使用的是v1还是v2
    + 调用`wd_get_accel_dev("zlib")` (v2 UADK接口)。若返回设备且`dev->flags & 0x1`,标记oww`HW_V2`
    + 否则调用`wd_get_available_den_num("zlib")`(v1接口会列出可用个数),若>0则标记为`HW_V1`
    + 否则`HW_NONE`，不启用硬件

### 入口路由
在deflate.c中
+ `deflateInit2_`:
  + 若`kz_get_devices()`为真，先调用`lz_deflateInit2_`(初始化软件state)，然后返回`kz_deflateInit2_`的结果(适配层做hw-specific init)，否则直接调用`lz_deflateInit2_`  // TODO: 存疑！
+ `deflate`: 若`kz_get_devices()`:
  + 调用`kz_deflate(strm, flush)`;`kz_deflate`根据平台返回
    + 一个硬件结果值(`Z_OK`, `Z_STREAM_END`, `Z_BUF_ERROR`等,或KAE自定义码)，
    + `Z_CALL_SOFT`(表示本次操作应由软件处理，如硬件条件不满足)
  + wrapper检查`kz_deflate`返回: 若是`Z_CALL_SOFT`则调用`lz_deflate`(软件实现)，否则调用硬件返回值
+ `deflateReset`/`deflateEnd`也有wrapper: 在有设备时先走`kz_deflateReset`/`kz_deflateEnd`,然后再走`lz_deflateReset`/`lz_deflateEnd`或按返回值决定

### kaezip适配层做路由
`kaezip_adapter.c`

+ `kz_deflateInit2_`, `kz_deflate`, `kz_deflateEnd`, `kz_deflateReset`是统一的适配入口
+ 首先调用`uadk_get_accel_platform()`来选定`g_platform`(`HW_NONE`/`HW_V1`/`HW_V2`)
+ 根据平台，选择调用
  + HW_V1 -> `kz_deflateInit2_v1`/`kz_deflate_v1`/`kz_deflateEnd_v1`等
  + HW_V2 -> `kz_deflate_init`/`kz_deflate_v2`/`kz_deflate_end`等
  + HW_NONE -> 返回指明使用软件(`Z_CALL_SOFT`或`Z_OK`,具体哪个取决于上下文),表示上层应走`lz_`

### v1(wrapdrive/hisi)实现要点
+ 在v1分支， 
+ `kz_deflateInit2_v1`:
  + 根据`windowBits`推断算法类型(zlib/gzip/raw)和是否支持，若不支持则把`kaezip_ctx`置0(表示不适用硬件)
  + 创建/获取`kaezip_ctx_t`(KAE上下文，包含header, buffer, 状态),并把它设置进`deflate_state`和`kaezip_ctx`字段(通过`setDeflateKaezipCtx(strm, (uLong)kaezip_ctx)`)
+ `kz_deflate_v1`:
  + 检查`kaezip_ctx`，准备格式头(zlib/gzip header), 并调用`kaezip_do_deflate`
    + `kaezip_do_deflate`会把`strm->next_in/avail_in`, `strm->next_out/avail_out`填到`kaezip_ctx`里，设置`kaezip_ctx->flush`等
    + 调用`kaezip_driver_do_comp(kaezip_ctx)`——此处为驱动接口，最终于wrapdrive交互（把数据送给硬件，等待并收回结果）
    + 返回后，使用`kaezip_ctx->consumed/produced`更新`z_stream`(next_in/next_out/avail_*/total_*)

### v2(UADK/wd)实现要点
+ v2使用UADK的`wd/wd_comp`接口，代码集中在`kaezip_comp.c`,`kaezip_init.c`, `kaezip_buffer.c`等
+ 初始化:`kz_deflate_init`->`kz_zlib_init`调用`wd_comp_init2_`(UADK初始化),按NUMA/设备分配设置上下文
+ 会为每个`z_stream`分配一个会话句柄(`wd_comp_alloc_sess`), 并把这个句柄放到`strm->reserved`字段(本来是zlib的state中的保留字段)
+ 执行压缩`kz_deflate_v2`->`kz_zlib_do_request_v2`:
  + 把`strm->next_in/avail_in`和`strm->next_out/avail_out`转换为`struct wd_comp_req req`(UADK请求结构)，并调用`kz_zlib_do_comp_implement`
  + `kz_zlib_do_comp_implement`可能分块(`INPUT_CHUNK_V2`/`OUTPUT_CHUNK_V2`),循环调用`wd_do_comp_strm(h_sess, &strm_req)`,处理硬件可能返回的分段(如果压缩后的块比dst空间打，会把多余量记录到`out_buffer->remained`),下一次deflate时先把remain数据拷回
  + 该分支顺用`strm->adler`字段保存`outbuffer_ptr`(一个用于保存overflow的输出缓冲对象),也是利用z_stream可复用的字段做“隐藏”缓冲
+ 释放: `kz_deflate_end` => `kz_zlib_uninit` => free session (`wd_comp_free_sess`), 并在最后做`wd_comp_uninit2()`

## 数据流
+ 应用/库调用zlib的`deflate(strm, flush)`
+ 新的`deflate` wrapper检查`CONF_KAEZIP`/`kz_get_devices()`:
  + 否: 直接调用`lz_deflate`(原生zlib)
  + 是: 调用kz_deflate(strm, flush)(适配层)
+ 在`kz_deflate`:
  + 通过`uadk_get_accel_platform()`判定平台(HW_V1/HW_V2/HW_NONE)
  + HW_NONE =>返回`Z_CALL_SOFT`(上层回退)，或直接`Z_OK`(基于实现)
  + HW_V1 => `getDeflateKaezipCtx(strm)`检查是否有上下文(若没有，可能`kz_deflateInit2_v1`未设置，返回软件降级),若存在则调用`kz_deflate_v1`
    + `kz_deflate_v1`会调用`kaezip_ztx`填充输入/输出指针于长度，并通过`kaezip_driver_do_comp`把任务扔给wrapdriver，结束后用`consumed/produced`更新strm状态
  + `HW_V2` => `kz_deflate_v2`调用`kz_zlib_do_request_v2`，构造`wd_comp_req`并调用`wd_do_comp_strm`，分块处理，更新`strm`
+ `kz_deflate`返回后，wrapper `deflate`会检查返回值
  + 如果返回`Z_CALL_SOFT`或某些错误代码 => 调用`lz_deflate`(软件实现)以保证兼容性
  + 否则直接返回`kz_deflate`的返回值给调用者

## 格式头/尾、缓冲和边界处理
+ 格式头(zlib/gzip): v1分支会在第一次输出之前拷贝格式头(`kaezip_deflate_set_fmt_header`),格式头大小通过`kaezzip_fmt_header_sz`获取。v2分支也通过`kz_outbuffer`机制管理尾部/超出数据
+ 格式尾(CRC/ISIZE): 在完成时`kaezip_set_fmt_tail`会拼接必要的尾数据，确保输出为合法的zlib/gzip留
+ 输出不足时(硬件产生的数据超过`avail_out`):
  + v2使用`out_buffer->remained`记录多余部分，下一次`deflate`时先把剩余数据拷回`strm->next_out`
  + v1使用`kaezip_ctx->end_block`/`remain`字段以类似的方式处理尾部
+ chunking: v2将大输入分解为INPUT_CHUNK_V2大小的块循环提交给硬件，每次调用要`wd_do_comp_strm`，硬件可能返回部分压缩数据，代码负责累积并拷回