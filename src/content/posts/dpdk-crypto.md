---
title: dpdk-crypto
published: 2025-11-15
description: 'dpdk crypto operations'
image: ''
tags: ['dpdk', 'crypto']
category: 'notes'
draft: false 
lang: ''
---

## operation processing
dpdk中的数据路径中的crypto op是通过以burst为中心的异步API实现的。

物理设备的enqueue burst API会把要处理的Ops放入硬件的Input queue中；虚拟设备在调用enqueue的时候就会去处理这些ops。

dequeue burst API会从crypto设备中取回所有已经处理过了的操作。对于物理设备而言，就是从设备处理过的queue中取出，对于虚拟设备而言是从一个`rte_ring`中取出处理后的ops。

### private data
对于session-based的op，set和get API提供了一个机制，让app可以在整个crypto session中，在private user data中存储和提取信息。

例如，假设一个app正在提交一个和一个session关联的op，并希望附带一些private user data，这些数据在crypto op执行完成后会被使用。在这样的情况下，就可以利用private data相关API来配置和获取userdata
```c
int rte_cryptodev_sym_session_set_user_data(
        struct rte_cryptodev_sym_session *sess, void *data, uint16_t size);

void * rte_cryptodev_sym_session_get_user_data(
        struct rte_cryptodev_sym_session *sess);
```

### user callback api
在每一次burst crypto ops 到/回 crypto device queue pair上之后回调用的回调函数。

### enqueue/dequeue burst api
略

### operation management and allocation
创建crypto op pool，为所有的crypto op分配一个内存池。
```c
struct rte_mempool *
rte_crypto_op_pool_create(const char *name, enum rte_crypto_op_type type,
                          unsigned nb_elts, unsigned cache_size, uint16_t priv_size,
                          int socket_id);
```
在创建pool的时候，回先调用`rte_crypto_op_init()`，再调用`__rte_crypto_op_reset()`复位。

`rte_crypto_op_alloc()`和`rte_crypto_op_bulk_alloc()`用来从已有的crypto op pool中分配出crypto op。调用这两个函数的时候，也一定回调用`__rte_crypto_op_reset()`，保证返回的op是初试状态。

```c
struct rte_crypto_op *rte_crypto_op_alloc(struct rte_mempool *mempool,
                                          enum rte_crypto_op_type type)

unsigned rte_crypto_op_bulk_alloc(struct rte_mempool *mempool,
                                  enum rte_crypto_op_type type,
                                  struct rte_crypto_op **ops, uint16_t nb_ops)
```
`rte_crypto_op_free`会把分配出来的op返回到Pool中。

## symmetric cryptography support
### session and session management
在对称密码处理中，session用于存储加密变换中定义的不可变数据，这些数据在数据包流的操作处理过程中被使用。
session用于管理诸如扩展后的加密密钥、HMAC 的 IPAD 和 OPAD 等信息，这些信息需要针对特定的加密操作进行计算，但在同一个数据流中，对于逐个数据包而言是不可变的。加密会话以底层 PMD最优的方式缓存这些不可变数据，从而进一步加速加密工作负载的卸载。

应用程序利用`rte_cryptodev_sym_session_pool_create()`来创建session mempool的header与private data，其中private data通过函数中的`elt_size`参数指定。session的private data是由驱动程序在加解密操作期间初始化和访问的。
为了获取一个crypto设备的session private data size,可以通过`rte_cryptodev_sym_get_private_session_size()`来获取。

当session mempools创建好了以后，调用`rte_cryptodev_sym_session_create()`来分配并创建一个session。创建出来的这个session仅能够被对应ID的设备使用（设备ID在调用的时候作为参数被传入）。另外，一个symmetric xform chain在创建的时候也被传入，标明操作和参数。

不再使用这个session的时候，用户调用`rte_cryptodev_sym_session_free()`释放session数据，把session返回到mempool中。

### transforms and transform chaining
symmetric crypto transforms(`rte_crypto_sym_xform`)用于指定crypto ops需要的细节。
"chaining"是因为还同时适用于，诸如cipher encrypt + auth generate的操作。`next`指针允许以上操作链式进行。
支持chaining操作的设备必须发布他支持的symmetric crypto op feature flag。
xform结构体的分配和赋值是在应用程序中做的。

### symmetric operation
symmetric crypto op(`rte_crypto_sym_op`)包含所有，与在引用mbuf上执行对称加解密相关的可变数据，用于cipher, auth, AEAD和链式操作。

一个最小的sym op必须由一个soruce data buffer(`m_src`)，一个valid session(或者是transform chain,如果是session-less模式)和至少要由的auth/cipher/AEAD参数。
