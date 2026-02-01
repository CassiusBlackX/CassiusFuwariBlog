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