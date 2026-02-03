---
title: openssl-async
published: 2026-02-02
description: 'examples using openssl async'
image: ''
tags: ["linux", "crypto"]
category: 'deployment'
draft: false 
lang: ''
---
# openssl async hardware simulation
```c
#include <errno.h>
#include <stdalign.h>
#include <stdatomic.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <fcntl.h>
#include <pthread.h>
#include <sys/select.h>
#include <unistd.h>

#include <openssl/async.h>
#include <openssl/crypto.h>
#include <openssl/err.h>

#define NUM_JOBS 3
#define HW_DELAY_SEC 2

typedef enum { HW_IDLE, HW_SUBMITTED, HW_COMPLETED, HW_ERROR } hw_status_t;

/* 作业上下文 (堆分配 + 引用计数) */
typedef struct {
  int job_id;
  unsigned char *input;
  size_t input_len;
  unsigned char *output;
  size_t output_len;
  _Atomic hw_status_t status;
  int pipe_read_fd;
  int pipe_write_fd;
  pthread_t thread_id;  // 线程ID (0=无效)
  _Atomic int refcount; // 引用计数 (作业函数+主程序)
  pthread_mutex_t lock; // 保护 cleanup 标志
  int cleanup_done;     // 防止重复清理
} hw_op_ctx_t;

/* 全局作业注册表 (确保主程序等待所有线程) */
static hw_op_ctx_t *global_ctxs[NUM_JOBS] = {NULL};
static pthread_mutex_t registry_lock = PTHREAD_MUTEX_INITIALIZER;

/* 安全释放上下文 (引用计数) */
static void ctx_release(hw_op_ctx_t *ctx) {
  if (!ctx)
    return;
  if (atomic_fetch_sub(&ctx->refcount, 1) == 1) {
    // 最后一个引用: 安全释放
    OPENSSL_free(ctx->input);
    OPENSSL_free(ctx->output);
    free(ctx);
    // printf("[Ctx] 作业上下文 %p 已安全释放\n", (void *)ctx);
  }
}

/* 增加引用 */
static void ctx_acquire(hw_op_ctx_t *ctx) {
  if (ctx)
    atomic_fetch_add(&ctx->refcount, 1);
}

/* 硬件模拟线程 (带错误处理) */
static void *hw_worker(void *arg) {
  hw_op_ctx_t *ctx = (hw_op_ctx_t *)arg;
  ctx_acquire(ctx); // 线程持有引用

  printf("[HW-%d] 硬件操作开始 (模拟 %d 秒延迟)...\n", ctx->job_id,
         HW_DELAY_SEC);
  sleep(HW_DELAY_SEC);

  // 模拟加密 (带输入验证)
  if (ctx->input && ctx->output && ctx->input_len > 0) {
    memcpy(ctx->output, ctx->input, ctx->input_len);
    for (size_t i = 0; i < ctx->input_len; i++) {
      ctx->output[i] ^= 0xAA;
    }
    ctx->output_len = ctx->input_len;
    atomic_store(&ctx->status, HW_COMPLETED);
    printf("[HW-%d] 操作成功完成!\n", ctx->job_id);
  } else {
    atomic_store(&ctx->status, HW_ERROR);
    fprintf(stderr, "[HW-%d] ERROR: 无效输入数据!\n", ctx->job_id);
  }

  // 通知主线程 (仅当 write_fd 有效)
  if (ctx->pipe_write_fd != -1) {
    char sig = 1;
    if (write(ctx->pipe_write_fd, &sig, 1) != 1) {
      if (errno != EPIPE) { // EPIPE 正常 (读端已关闭)
        perror("[HW] 管道写入警告");
      }
    }
    // 注意: 不关闭 write_fd (由作业函数清理)
  }

  // 注销全局注册 (线程结束)
  pthread_mutex_lock(&registry_lock);
  if (global_ctxs[ctx->job_id] == ctx) {
    global_ctxs[ctx->job_id] = NULL;
  }
  pthread_mutex_unlock(&registry_lock);

  ctx_release(ctx); // 释放线程持有的引用
  return NULL;
}

/* ASYNC 作业核心函数 (统一清理路径) */
static int async_encrypt_job(void *arg) {
  hw_op_ctx_t *ctx = (hw_op_ctx_t *)arg;
  if (!ctx)
    return 0;

  ctx_acquire(ctx); // 作业函数持有引用
  int ret = 0;      // 默认失败

  ASYNC_JOB *job = ASYNC_get_current_job();
  if (!job) {
    fprintf(stderr, "[Job-%d] FATAL: ASYNC_get_current_job() = NULL!\n",
            ctx->job_id);
    goto cleanup;
  }

  ASYNC_WAIT_CTX *waitctx = ASYNC_get_wait_ctx(job);
  if (!waitctx) {
    fprintf(stderr, "[Job-%d] FATAL: ASYNC_get_wait_ctx() = NULL!\n",
            ctx->job_id);
    goto cleanup;
  }

  hw_status_t current_status = atomic_load(&ctx->status);
  printf("[Job-%d] ENTRY | status=%d | waitctx=%p\n", ctx->job_id,
         current_status, (void *)waitctx);

  // 情况1: 硬件已完成 (恢复执行)
  if (current_status == HW_COMPLETED) {
    printf("[Job-%d] 硬件操作已完成，返回成功\n", ctx->job_id);
    ret = 1;
    goto cleanup;
  }

  // 情况2: 首次调用，提交硬件操作
  if (current_status == HW_IDLE) {
    printf("[Job-%d] 首次调用: 创建管道并提交硬件...\n", ctx->job_id);

    int pipefd[2];
    if (pipe(pipefd) == -1) {
      perror("[Job] pipe 创建失败");
      goto cleanup;
    }

    // 设置读端非阻塞
    int flags = fcntl(pipefd[0], F_GETFL, 0);
    fcntl(pipefd[0], F_SETFL, flags | O_NONBLOCK);

    ctx->pipe_read_fd = pipefd[0];
    ctx->pipe_write_fd = pipefd[1];
    atomic_store(&ctx->status, HW_SUBMITTED);

    // 【关键】cleanup 回调必须为 NULL!
    if (!ASYNC_WAIT_CTX_set_wait_fd(waitctx, ctx, pipefd[0], NULL, NULL)) {
      fprintf(stderr, "[Job-%d] ASYNC_WAIT_CTX_set_wait_fd 失败!\n",
              ctx->job_id);
      close(pipefd[0]);
      close(pipefd[1]);
      ctx->pipe_read_fd = ctx->pipe_write_fd = -1;
      goto cleanup;
    }

    printf("[Job-%d] 管道创建成功 | read_fd=%d | write_fd=%d\n", ctx->job_id,
           pipefd[0], pipefd[1]);

    // 启动硬件线程 (不 detach!)
    if (pthread_create(&ctx->thread_id, NULL, hw_worker, ctx) != 0) {
      fprintf(stderr, "[Job-%d] pthread_create 失败!\n", ctx->job_id);
      close(pipefd[0]);
      close(pipefd[1]);
      ctx->pipe_read_fd = ctx->pipe_write_fd = -1;
      goto cleanup;
    }

    // 注册到全局表 (主程序退出前等待)
    pthread_mutex_lock(&registry_lock);
    global_ctxs[ctx->job_id] = ctx;
    pthread_mutex_unlock(&registry_lock);

    printf("[Job-%d] >>> 调用 ASYNC_pause_job() 让出控制权 <<<\n", ctx->job_id);
    ASYNC_pause_job(); // 暂停点 (恢复后从此处继续)

    printf("[Job-%d] RESUMED | status=%d\n", ctx->job_id,
           atomic_load(&ctx->status));
  }

  // 情况3: 恢复执行，处理完成事件
  if (atomic_load(&ctx->status) == HW_SUBMITTED) {
    printf("[Job-%d] 检查管道事件...\n", ctx->job_id);

    if (ctx->pipe_read_fd != -1) {
      char dummy;
      ssize_t n = read(ctx->pipe_read_fd, &dummy, 1);
      if (n > 0) {
        printf("[Job-%d] 读取到完成信号 (n=%zd)\n", ctx->job_id, n);
      } else if (n == -1 && errno == EAGAIN) {
        printf("[Job-%d] 管道无数据 (EAGAIN)，重新暂停...\n", ctx->job_id);
        ASYNC_pause_job(); // 再次暂停 (不会执行 cleanup)
                           // 恢复后继续执行
      } else {
        fprintf(stderr, "[Job-%d] 管道读取异常 (n=%zd, errno=%d)\n",
                ctx->job_id, n, errno);
        goto cleanup;
      }
    }

    // 验证最终状态
    hw_status_t final_status = atomic_load(&ctx->status);
    if (final_status == HW_COMPLETED) {
      printf("[Job-%d] 状态验证通过: HW_COMPLETED\n", ctx->job_id);
      ret = 1;
    } else if (final_status == HW_ERROR) {
      fprintf(stderr, "[Job-%d] 硬件报告错误状态!\n", ctx->job_id);
      ret = 0;
    } else {
      fprintf(stderr, "[Job-%d] 状态错误! 期望 HW_COMPLETED, 实际=%d\n",
              ctx->job_id, final_status);
      ret = 0;
    }
  }

cleanup:
  // 【关键】线程安全清理 (防止重复清理)
  pthread_mutex_lock(&ctx->lock);
  if (!ctx->cleanup_done) {
    ctx->cleanup_done = 1;

    // 清理管道 fd
    if (ctx->pipe_read_fd != -1) {
      close(ctx->pipe_read_fd);
      ctx->pipe_read_fd = -1;
    }
    if (ctx->pipe_write_fd != -1) {
      close(ctx->pipe_write_fd);
      ctx->pipe_write_fd = -1;
    }

    // 【关键】等待硬件线程结束 (避免访问已销毁内存)
    if (ctx->thread_id != 0) {
      printf("[Job-%d] 等待硬件线程结束...\n", ctx->job_id);
      pthread_join(ctx->thread_id, NULL);
      ctx->thread_id = 0;
      printf("[Job-%d] 硬件线程已安全 join\n", ctx->job_id);
    }

    pthread_mutex_unlock(&ctx->lock);

    if (ret == 1) {
      printf("[Job-%d] 作业成功完成!\n", ctx->job_id);
    } else {
      printf("[Job-%d] 作业失败 (结果=%d)\n", ctx->job_id, ret);
    }
  } else {
    pthread_mutex_unlock(&ctx->lock);
  }

  ctx_release(ctx); // 释放作业函数持有的引用
  return ret;
}

/* 事件循环 (简化版) */
static void run_event_loop(ASYNC_JOB **jobs, int num_jobs) {
  fd_set readfds;
  int max_fd = -1;
  struct timeval tv = {.tv_sec = 1, .tv_usec = 0};
  FD_ZERO(&readfds);

  for (int i = 0; i < num_jobs; i++) {
    if (!jobs[i])
      continue;
    ASYNC_WAIT_CTX *wctx = ASYNC_get_wait_ctx(jobs[i]);
    if (!wctx)
      continue;

    size_t num_fds = 0;
    if (!ASYNC_WAIT_CTX_get_all_fds(wctx, NULL, &num_fds) || num_fds == 0)
      continue;

    OSSL_ASYNC_FD *fds = OPENSSL_zalloc(num_fds * sizeof(OSSL_ASYNC_FD));
    if (!fds)
      continue;

    if (ASYNC_WAIT_CTX_get_all_fds(wctx, fds, &num_fds)) {
      for (size_t j = 0; j < num_fds; j++) {
        if (fds[j] != OSSL_BAD_ASYNC_FD) {
          FD_SET(fds[j], &readfds);
          if ((int)fds[j] > max_fd)
            max_fd = fds[j];
        }
      }
    }
    OPENSSL_free(fds);
  }

  if (max_fd == -1)
    return;
  select(max_fd + 1, &readfds, NULL, NULL, &tv);
}

/* 主程序退出前等待所有硬件线程 */
static void wait_all_hw_threads() {
  printf("\n[Main] 检查残留硬件线程...\n");
  for (int i = 0; i < NUM_JOBS; i++) {
    pthread_mutex_lock(&registry_lock);
    hw_op_ctx_t *ctx = global_ctxs[i];
    pthread_mutex_unlock(&registry_lock);

    if (ctx) {
      printf("[Main] 等待作业 %d 的硬件线程结束...\n", i);
      // 作业函数会 join 线程，但双重保险
      if (ctx->thread_id != 0) {
        pthread_join(ctx->thread_id, NULL);
      }
      ctx_release(ctx); // 释放注册表持有的引用
    }
  }
  printf("[Main] 所有硬件线程已安全结束\n");
}

int main() {
  if (!ASYNC_init_thread(10, 10)) {
    fprintf(stderr, "ASYNC_init_thread 失败!\n");
    return 1;
  }

  printf("========================================\n");
  printf("OpenSSL ASYNC 机制演示\n");
  printf("OpenSSL 版本: %s\n", OPENSSL_VERSION_TEXT);
  printf("========================================\n\n");

  // 初始化全局注册表
  memset(global_ctxs, 0, sizeof(global_ctxs));

  // 【关键】为每个作业创建堆分配的上下文
  hw_op_ctx_t *ctxs[NUM_JOBS] = {NULL};
  int results[NUM_JOBS] = {0};
  ASYNC_JOB *jobs[NUM_JOBS] = {NULL};
  ASYNC_WAIT_CTX *job_waitctxs[NUM_JOBS] = {NULL};
  int active = NUM_JOBS;

  for (int i = 0; i < NUM_JOBS; i++) {
    ctxs[i] = (hw_op_ctx_t *)calloc(1, sizeof(hw_op_ctx_t));
    if (!ctxs[i]) {
      fprintf(stderr, "内存分配失败!\n");
      goto cleanup;
    }

    // 初始化上下文
    ctxs[i]->job_id = i;
    atomic_init(&ctxs[i]->status, HW_IDLE);
    ctxs[i]->pipe_read_fd = ctxs[i]->pipe_write_fd = -1;
    ctxs[i]->thread_id = 0;
    atomic_init(&ctxs[i]->refcount, 1); // 初始引用: 主程序持有
    pthread_mutex_init(&ctxs[i]->lock, NULL);
    ctxs[i]->cleanup_done = 0;

    // 设置输入数据
    const char *msg = "ASYNC-OK";
    ctxs[i]->input_len = strlen(msg);
    ctxs[i]->input = OPENSSL_malloc(ctxs[i]->input_len);
    ctxs[i]->output = OPENSSL_malloc(ctxs[i]->input_len + 1);
    memcpy(ctxs[i]->input, msg, ctxs[i]->input_len);

    // 创建 WAIT_CTX
    job_waitctxs[i] = ASYNC_WAIT_CTX_new();
    if (!job_waitctxs[i]) {
      fprintf(stderr, "创建 WAIT_CTX 失败!\n");
      goto cleanup;
    }

    printf("[Main] 初始化作业 %d: ctx=%p, WAIT_CTX=%p\n", i, (void *)ctxs[i],
           (void *)job_waitctxs[i]);
  }

  // 主处理循环
  printf("\n[Main] 开始作业处理循环...\n");
  while (active > 0) {
    for (int i = 0; i < NUM_JOBS; i++) {
      if (jobs[i] == NULL && active <= 0)
        continue;

      int state =
          ASYNC_start_job(&jobs[i], job_waitctxs[i], &results[i],
                          async_encrypt_job, ctxs[i], sizeof(hw_op_ctx_t));

      switch (state) {
      case ASYNC_FINISH:
        printf("[Main] 作业 %d: ASYNC_FINISH (结果=%d)\n", i, results[i]);
        if (results[i] == 1) {
          printf(">>> 作业 %d 成功! 输出: ", i);
          for (size_t j = 0; j < ctxs[i]->output_len; j++) {
            printf("%02x ", ctxs[i]->output[j]);
          }
          printf("\n");
        }
        // 释放 WAIT_CTX (作业生命周期结束)
        if (job_waitctxs[i]) {
          ASYNC_WAIT_CTX_free(job_waitctxs[i]);
          job_waitctxs[i] = NULL;
        }
        jobs[i] = NULL;
        active--;
        // 主程序释放对 ctx 的引用
        ctx_release(ctxs[i]);
        break;

      case ASYNC_PAUSE:
        // 正常: 等待事件
        break;

      case ASYNC_ERR:
        fprintf(stderr, "[Main] 作业 %d: ASYNC_ERR (%s)\n", i,
                ERR_error_string(ERR_get_error(), NULL));
        // NOTE:
        // fallthrough
      case ASYNC_NO_JOBS:
        if (state == ASYNC_NO_JOBS) {
          printf("[Main] 作业 %d: ASYNC_NO_JOBS\n", i);
        }
        // 清理资源
        if (job_waitctxs[i]) {
          ASYNC_WAIT_CTX_free(job_waitctxs[i]);
          job_waitctxs[i] = NULL;
        }
        jobs[i] = NULL;
        active--;
        ctx_release(ctxs[i]);
        break;
      }
    }

    if (active > 0) {
      run_event_loop(jobs, NUM_JOBS);
    }
  }

  // 【关键】等待所有硬件线程结束 (防止主程序退出后线程访问内存)
  wait_all_hw_threads();

cleanup:
  // 释放剩余资源 (正常流程应无剩余)
  for (int i = 0; i < NUM_JOBS; i++) {
    if (job_waitctxs[i])
      ASYNC_WAIT_CTX_free(job_waitctxs[i]);
    if (ctxs[i]) {
      // 确保线程已结束
      if (ctxs[i]->thread_id != 0) {
        pthread_join(ctxs[i]->thread_id, NULL);
      }
      pthread_mutex_destroy(&ctxs[i]->lock);
      // ctx_release 会处理最终释放
      ctx_release(ctxs[i]);
    }
  }

  ASYNC_cleanup_thread();
  printf("\n========================================\n");
  printf("✓ 程序安全退出 (无内存泄漏/段错误)\n");
  printf("========================================\n");
  return 0;
}
```


# 前后端分离的openssl加解密任务异步卸载
基于DPDK共享内存，实现前后端分离的openssl ASYNC方式的加解密任务异步卸载的模拟。
## 公共头文件
```c
// common.h
#ifndef CRYPTO_ASYNC_COMMON_H
#define CRYPTO_ASYNC_COMMON_H

#include <stdarg.h>
#include <stddef.h>

#include <rte_common.h>
#include <rte_mempool.h>
#include <rte_ring.h>
#include <stdio.h>

#define MEMPOOL_NAME "MSG_POOL"
#define MSG_STRUCT_POOL "MSG_STRUCT_POOL"
#define RING_FE_TO_BE "FE_TO_BE"
#define RING_BE_TO_FE "BE_TO_FE"
#define MP_MSG_NAME "fd_sync"

#define MAX_JOBS 4
#define MAX_DATA_LEN 1024
#define POOL_SIZE 1024
#define CACHE_SIZE 0

typedef struct {
  void *data;
  size_t len;
  int job_id;
} task_msg;

static inline void _log_error(const char *fe_be, const char *func, int line,
                              const char *fmt, ...)
    __attribute__((format(printf, 4, 5)));

static inline void _log_error(const char *fe_be, const char *func, int line,
                              const char *fmt, ...) {
  fprintf(stderr, "[%s]ERROR(%s:%d):", fe_be, func, line);
  va_list args;
  va_start(args, fmt);
  vfprintf(stderr, fmt, args);
  va_end(args);
  fprintf(stderr, "\n");
}

static inline void _log_info(const char *fe_be, const char *func, int line,
                              const char *fmt, ...)
    __attribute__((format(printf, 4, 5)));

static inline void _log_info(const char *fe_be, const char *func, int line,
                              const char *fmt, ...) {
  fprintf(stdout, "[%s]INFO(%s:%d):", fe_be, func, line);
  va_list args;
  va_start(args, fmt);
  vfprintf(stdout, fmt, args);
  va_end(args);
  fprintf(stdout, "\n");
}

#ifndef FE_BE
#error "FE_BE must be defined before including common.h"
#endif // FE_BE
#define log_error(fmt, ...)                                                    \
  _log_error(FE_BE, __func__, __LINE__, fmt, ##__VA_ARGS__)
#define log_info(fmt, ...)                                                    \
  _log_info(FE_BE, __func__, __LINE__, fmt, ##__VA_ARGS__)

#endif // CRYPTO_ASYNC_COMMON_H
```

## 前端
```c
#include <errno.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <sys/epoll.h>
#include <sys/eventfd.h>
#include <unistd.h>

#include <openssl/async.h>

#include <rte_eal.h>
#include <rte_mempool.h>
#include <rte_ring.h>

#define FE_BE "frontend"
#include "common.h"

#define MAX_EVENTS 16

static struct rte_mempool *mp_data = NULL;
static struct rte_mempool *mp_struct = NULL;
static struct rte_ring *fe_to_be = NULL;
static struct rte_ring *be_to_fe = NULL;

typedef struct {
  int id;
  ASYNC_JOB *job;
  ASYNC_WAIT_CTX *waitctx;
  int evfd;
  task_msg *msg;
  struct rte_ring *fe_to_be;
  struct rte_ring *be_to_fe;
  struct rte_mempool *mp_data;
  struct rte_mempool *mp_struct;
} job_ctx;

static int async_job_func(void *arg) {
  job_ctx *ctx = (job_ctx *)arg;
  task_msg *msg = ctx->msg;
  if (!ctx || !msg) {
    log_error("got NULL in ctx(%p) or msg(%p)", ctx, msg);
    return -1;
  }

  // prepare data
  uint8_t *data_ptr = NULL;
  if (rte_mempool_get(ctx->mp_data, (void **)&data_ptr) < 0) {
    log_error("failed to get in mp_data");
    return -1;
  }

  msg->data = data_ptr;
  int len = snprintf((char *)data_ptr, MAX_DATA_LEN, "hello from frontend: %d",
                     ctx->id);
  if (len < 0) {
    log_error("failed to fill in data_ptr");
  }
  msg->len = len + 1;
  msg->job_id = ctx->id;
  log_info("job-%d: sending task, msg_addr: %p, data[0]=%#x\n", ctx->id, msg, 
            data_ptr[0]);

  // send to backend
  if (rte_ring_enqueue(ctx->fe_to_be, msg) < 0) {
    log_error("failed to enqueue to be");
    rte_mempool_put(ctx->mp_data, data_ptr);
    return -1;
  }
  log_info("job-%d: enqueue done, pause", ctx->id);
  ASYNC_pause_job();
  log_info("job-%d: resumed, result data[0]=%#x", ctx->id, data_ptr[0]);

  // NOTE: the following steps are useless!
  // wait for result msg to return (optional, just to cleanup)
  task_msg *tmp_msg = NULL;
  while (rte_ring_dequeue(ctx->be_to_fe, (void **)&tmp_msg) != 0) {
    rte_pause();
  }
  uint8_t *tmp_ptr = tmp_msg->data;
  log_info("job-%d: got msg: %p, data[0]: %#x", ctx->id, tmp_msg, tmp_ptr[0]);

  // free resource
  rte_mempool_put(ctx->mp_data, data_ptr);
  return 0;
}

// send eventfd to primary proc
static void sync_evfd_to_primary(int job_id, int evfd) {
  struct rte_mp_msg msg;
  memset(&msg, 0, sizeof(msg));
  snprintf(msg.name, RTE_MP_MAX_NAME_LEN, "%s", MP_MSG_NAME);
  msg.len_param = sizeof(int);
  msg.num_fds = 1;
  *(int *)msg.param = job_id;
  msg.fds[0] = evfd;
  rte_mp_sendmsg(&msg);
}

int main(int argc, char **argv) {
  int ret, epfd;
  job_ctx jobs[MAX_JOBS] = {0};
  int job_rets[MAX_JOBS] = {0};
  memset(jobs, 0, sizeof(jobs));
  memset(job_rets, 0, sizeof(job_rets));

  ret = rte_eal_init(argc, argv);
  if (ret < 0) {
    log_error("eal init failed");
    exit(1);
  }

  mp_data = rte_mempool_lookup(MEMPOOL_NAME);
  mp_struct = rte_mempool_lookup(MSG_STRUCT_POOL);
  fe_to_be = rte_ring_lookup(RING_FE_TO_BE);
  be_to_fe = rte_ring_lookup(RING_BE_TO_FE);

  if (!mp_data || !mp_struct || !fe_to_be || !be_to_fe) {
    log_error("failed to lookup mempool/ring. mp_data: %p, mp_struct: %p, "
              "fe_to_be: %p, be_to_fe: %p",
              mp_data, mp_struct, fe_to_be, be_to_fe);
    exit(1);
  }

  epfd = epoll_create1(0);
  if (epfd < 0) {
    log_error("failed to create epoll fd");
    exit(1);
  }

  // initialize jobs
  for (int i = 0; i < MAX_JOBS; i++) {
    jobs[i].id = i;
    jobs[i].job = NULL;
    jobs[i].mp_data = mp_data;
    jobs[i].mp_struct = mp_struct;
    jobs[i].fe_to_be = fe_to_be;
    jobs[i].be_to_fe = be_to_fe;

    // allocate msg struct from mempool
    if (rte_mempool_get(mp_struct, (void **)&jobs[i].msg) < 0) {
      log_error("failed to alloc for jobs[%d].msg", i);
      goto cleanup;
    }

    jobs[i].evfd = eventfd(0, EFD_NONBLOCK);
    if (jobs[i].evfd < 0) {
      log_error("failed to create evfd for jobs[%d]", i);
      goto cleanup;
    }

    jobs[i].waitctx = ASYNC_WAIT_CTX_new();
    ASYNC_WAIT_CTX_set_wait_fd(jobs[i].waitctx, jobs[i].waitctx, jobs[i].evfd,
                               NULL, NULL);

    // send fd to primary
    sync_evfd_to_primary(i, jobs[i].evfd);

    struct epoll_event ev = {
        .events = EPOLLIN,
        .data.ptr = &jobs[i],
    };
    epoll_ctl(epfd, EPOLL_CTL_ADD, jobs[i].evfd, &ev);

    ASYNC_start_job(&jobs[i].job, jobs[i].waitctx, &job_rets[i], async_job_func,
                    &jobs[i], sizeof(jobs[i]));
  }
  log_info("frontend started, waiting for events...");

  int active_jobs = MAX_JOBS;
  struct epoll_event events[MAX_EVENTS];
  while (active_jobs > 0) {
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
    if (nfds < -1) {
      log_error("error in epoll_wait, err: %s", strerror(errno));
    }

    for (int i = 0; i < nfds; i++) {
      job_ctx *ctx = (job_ctx *)events[i].data.ptr;
      uint64_t val;
      read(ctx->evfd, &val, sizeof(val));

      int job_ret;
      ret = ASYNC_start_job(&ctx->job, ctx->waitctx, &job_ret, async_job_func,
                            ctx, sizeof(*ctx));

      switch (ret) {
      case ASYNC_FINISH:
        active_jobs--;
        rte_mempool_put(mp_struct, ctx->msg);
        log_info("job: %d done, job_ret: %d", ctx->id, job_ret);
        break;
      case ASYNC_NO_JOBS:
        log_info("got no job in :%d", ctx->id);
        break;
      case ASYNC_ERR:
        log_error("got error in %d", ctx->id);
        break;
      case ASYNC_PAUSE:
        log_info("got pause in %d", ctx->id);
        break;
      }
    }
  }

  log_info("all jobs done, frontend exiting...");
  rte_eal_cleanup();

cleanup:
  return 0;
}
```

## 后端
```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <signal.h>
#include <unistd.h>

#include <rte_eal.h>
#include <rte_errno.h>
#include <rte_lcore.h>
#include <rte_ring_core.h>
#include <rte_mempool.h>
#include <rte_ring.h>

#define FE_BE "backend"
#include "common.h"

static sig_atomic_t quit = 0;
static int job_evfds[MAX_JOBS] = {0};

static struct rte_mempool *mp_data = NULL;
static struct rte_mempool *mp_struct = NULL;
static struct rte_ring *fe_to_be = NULL;
static struct rte_ring *be_to_fe = NULL;

void sigint_handler(int id) {
  if (id == SIGINT || id == SIGTERM) {
    quit = 1;
  }
}

static int handle_fd_sync(const struct rte_mp_msg *msg,
                          const void *peer __attribute__((unused))) {
  int job_id = *(int *)msg->param;
  if (job_id >= 0 && job_id < MAX_JOBS) {
    job_evfds[job_id] = msg->fds[0];
    log_info("received evfd %d for job %d", job_evfds[job_id], job_id);
  } else {
    log_error("received evfd %d for job %d", job_evfds[job_id], job_id);
  }
  return 0;
}

int main(int argc, char **argv) {
  int ret;
  ret = rte_eal_init(argc, argv);
  if (ret < 0) {
    log_error("eal init failed");
    exit(1);
  }
  signal(SIGINT, sigint_handler);
  signal(SIGTERM, sigint_handler);
  for (int i = 0; i < MAX_JOBS; i++) {
    job_evfds[i] = -1;
  }

  rte_mp_action_register(MP_MSG_NAME, handle_fd_sync);

  // create mempool
  mp_data =
      rte_mempool_create(MEMPOOL_NAME, POOL_SIZE, MAX_DATA_LEN, CACHE_SIZE, 0,
                         NULL, NULL, NULL, NULL, rte_socket_id(), 0);
  mp_struct = rte_mempool_create(MSG_STRUCT_POOL, POOL_SIZE, sizeof(task_msg),
                                 CACHE_SIZE, 0, NULL, NULL, NULL, NULL,
                                 rte_socket_id(), 0);

  if (mp_data == NULL) {
    log_error("mp_data mempool create failed");
    exit(1);
  }
  if (mp_struct == NULL) {
    log_error("mp_struct mempool create failed");
    exit(1);
  }

  // create ring
  fe_to_be =
      rte_ring_create(RING_FE_TO_BE, 1024, rte_socket_id(), RING_F_SC_DEQ);
  be_to_fe =
      rte_ring_create(RING_BE_TO_FE, 1024, rte_socket_id(), RING_F_SC_DEQ);
  if (!fe_to_be || !be_to_fe) {
    log_error("failed to create ring");
    goto cleanup;
  }
  log_info("dpdk resource prepared success, polling tasks...");

  while (!quit) {
    task_msg *msg = NULL;
    if (rte_ring_dequeue(fe_to_be, (void **)&msg) == 0) {
      // dequeued msg from frontend
      uint8_t *p = (uint8_t *)msg->data;
      if (p) {
        for (size_t i = 0; i < msg->len; i++) {
          // bitwise NOT to simulate encrypt process
          p[i] = ~p[i];
        }
      } else {
        log_error("null ptr in msg");
      }

      // sleep a while to pretend hardware processing
      usleep(100000 * (rand() % 10));

      // notify frontend using the stored eventfd
      if (msg->job_id >= 0 && msg->job_id < MAX_JOBS &&
          job_evfds[msg->job_id] > 0) {
        uint64_t val = 1;
        if (write(job_evfds[msg->job_id], &val, sizeof(val)) != sizeof(val)) {
          perror("write evfd");
        } else {
          log_info("write evfd: %d to wake up frontend job_id: %d",
                   job_evfds[msg->job_id], msg->job_id);
        }
      }

      // optional: enqueue msg ptr backend to frontend
      // this step is useless, because frontend itself can have access to msg
      rte_ring_enqueue(be_to_fe, msg);
    }
  }

  log_info("receive sigint/sigterm, shutting down...");

cleanup:
  if (mp_data)
    rte_mempool_free(mp_data);
  if (mp_struct)
    rte_mempool_free(mp_struct);
  if (fe_to_be)
    rte_ring_free(fe_to_be);
  if (be_to_fe)
    rte_ring_free(be_to_fe);
  rte_eal_cleanup();
  return 0;
}

```