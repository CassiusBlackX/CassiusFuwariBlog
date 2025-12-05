---
title: dpdk-do-sym-crypto
published: 2025-12-05
description: 'a sample code to perform a dpdk sym crypto op'
image: ''
tags: ['dpdk', 'crypto']
category: deployment
draft: false 
lang: ''
---
# 一个dpdk执行crypto的样例程序

## 代码
```c
#include <assert.h>
#include <rte_config.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <rte_crypto.h>
#include <rte_crypto_sym.h>
#include <rte_cryptodev.h>
#include <rte_eal.h>
#include <rte_lcore.h>
#include <rte_mbuf.h>
#include <rte_mbuf_core.h>
#include <rte_mempool.h>

#define CRYPTO_OP_POOL_NAME "crypto_op_pool"
#define MBUF_POOL_NAME "data_mbuf_pool"
#define SESS_POOL_NAME "sym_sess_pool"
#define MBUFS_PER_POOL 2048
#define CRYPTO_OPS_PER_POOL 2048
#define SESS_PER_POOL 2048

#define AES_KEY_SIZE 32 // 256 bits
#define IV_SIZE 16      // 128 bits
#define BLOCK_SIZE 16   // AES block size
#define MAX_SESSIONS 10
#define PMD_NAME "crypto_openssl"

static struct rte_mempool *crypto_op_pool = NULL;
static struct rte_mempool *mbuf_pool = NULL;
static struct rte_mempool *sess_pool = NULL;

static uint8_t aes_key[AES_KEY_SIZE] = {
    0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08, 0x09, 0x0a,
    0x0b, 0x0c, 0x0d, 0x0e, 0x0f, 0x10, 0x11, 0x12, 0x13, 0x14, 0x15,
    0x16, 0x17, 0x18, 0x19, 0x1a, 0x1b, 0x1c, 0x1d, 0x1e, 0x1f};
static uint8_t iv_data[IV_SIZE] = {0x01, 0x02, 0x03, 0x04, 0x05, 0x06,
                                   0x07, 0x08, 0x09, 0x0a, 0x0b, 0x0c,
                                   0x0d, 0x0e, 0x0f, 0x10};

int setup_cryptodev() {
  uint8_t cdev_ids[64] = {0};
  int num_cdevs = rte_cryptodev_devices_get(PMD_NAME, cdev_ids, 64);
  assert(num_cdevs > 0 && "no cdev " PMD_NAME " found!");
  uint8_t cdev_id = cdev_ids[0];
  struct rte_cryptodev_config config = {
      .nb_queue_pairs = 1,
      .socket_id = rte_socket_id(),
  };
  if (rte_cryptodev_configure(cdev_id, &config) < 0) {
    fprintf(stderr, "failed to configure crypto dev %u\n", cdev_id);
    return -1;
  }

  struct rte_cryptodev_qp_conf qp_conf = {
      .nb_descriptors = 2048,
  };
  if (rte_cryptodev_queue_pair_setup(cdev_id, 0, &qp_conf, rte_socket_id()) <
      0) {
    fprintf(stderr, "failed to setup queue pair 0 on device %u\n", cdev_id);
    return -1;
  }

  if (rte_cryptodev_start(cdev_id) < 0) {
    fprintf(stderr, "failed to start crypto dev %u\b", cdev_id);
    return -1;
  }
  return cdev_id;
}

struct rte_crypto_sym_session *create_sym_encrypt_session(uint8_t cdev_id) {
  struct rte_crypto_cipher_xform cipher = {
      .algo = RTE_CRYPTO_CIPHER_AES_CBC,
      .op = RTE_CRYPTO_CIPHER_OP_ENCRYPT,
      .key.length = AES_KEY_SIZE,
      .key.data = aes_key,
      .iv.length = IV_SIZE,
      .iv.offset =
          sizeof(struct rte_crypto_op) + sizeof(struct rte_crypto_sym_op),
  };

  struct rte_crypto_sym_xform xform = {
      .next = NULL,
      .type = RTE_CRYPTO_SYM_XFORM_CIPHER,
      .cipher = cipher,
  };

  struct rte_crypto_sym_session *sess =
      rte_cryptodev_sym_session_create(cdev_id, &xform, sess_pool);
  if (!sess) {
    fprintf(stderr, "failed to create encrypt session for cdev :%u\n", cdev_id);
    return NULL;
  }
  return sess;
}

struct rte_crypto_sym_session *create_sym_decrypt_session(uint8_t cdev_id) {
  struct rte_crypto_cipher_xform cipher = {
      .algo = RTE_CRYPTO_CIPHER_AES_CBC,
      .op = RTE_CRYPTO_CIPHER_OP_DECRYPT,
      .key.length = AES_KEY_SIZE,
      .key.data = aes_key,
      .iv.length = IV_SIZE,
      .iv.offset =
          sizeof(struct rte_crypto_op) + sizeof(struct rte_crypto_sym_op),
  };

  struct rte_crypto_sym_xform xform = {
      .next = NULL,
      .type = RTE_CRYPTO_SYM_XFORM_CIPHER,
      .cipher = cipher,
  };

  struct rte_crypto_sym_session *sess =
      rte_cryptodev_sym_session_create(cdev_id, &xform, sess_pool);
  if (!sess) {
    fprintf(stderr, "failed to create decrypt session for cdev :%u\n", cdev_id);
    return NULL;
  }
  return sess;
}

static const char *rte_crypto_op_err_msg(const struct rte_crypto_op *op) {
  char *status_str;
  switch (op->status) {
  case RTE_CRYPTO_OP_STATUS_NOT_PROCESSED:
    status_str = "RTE_CRYPTO_OP_STATUS_NOT_PROCESSED";
    break;
  case RTE_CRYPTO_OP_STATUS_AUTH_FAILED:
    status_str = "RTE_CRYPTO_OP_STATUS_AUTH_FAILED";
    break;
  case RTE_CRYPTO_OP_STATUS_INVALID_SESSION:
    status_str = "RTE_CRYPTO_OP_STATUS_INVALID_SESSION";
    break;
  case RTE_CRYPTO_OP_STATUS_INVALID_ARGS:
    status_str = "RTE_CRYPTO_OP_STATUS_INVALID_ARGS";
    break;
  case RTE_CRYPTO_OP_STATUS_ERROR:
    status_str = "RTE_CRYPTO_OP_STATUS_ERROR";
    break;
  default:
    printf("rte_crypto_op_err_msg, but no error\n");
    status_str = NULL;
  }
  return status_str;
}

bool do_encrypt(uint8_t cdev_id, struct rte_crypto_sym_session *sess,
                const char *plaintext, char **encrypted_text,
                size_t *encrypted_len) {
  bool success = false;
  size_t data_len = strlen(plaintext);
  uint8_t padding_len = BLOCK_SIZE - (data_len % BLOCK_SIZE);
  size_t total_len = data_len + padding_len;

  // 1. alloc mbuf
  struct rte_mbuf *m = rte_pktmbuf_alloc(mbuf_pool);
  if (!m) {
    fprintf(stderr, "failed to alloc for mbuf\n");
    goto cleanup;
  }

  // 2. fill in data
  if (rte_pktmbuf_append(m, total_len) == 0) {
    fprintf(stderr, "mbuf append failed\n");
    rte_pktmbuf_free(m);
    return false;
  }
  uint8_t *data_ptr = rte_pktmbuf_mtod(m, uint8_t *);
  memcpy(data_ptr, plaintext, data_len);
  memset(data_ptr + data_len, 0, padding_len);

  printf("Original plaintext (padded): '");
  for (size_t i = 0; i < total_len; i++) {
    printf("%02x ", data_ptr[i]);
  }
  printf("\n");
  printf("Data length: %zu (Padded: %zu)\n", data_len, total_len);

  // 3. alloc crypto op
  struct rte_crypto_op *op =
      rte_crypto_op_alloc(crypto_op_pool, RTE_CRYPTO_OP_TYPE_SYMMETRIC);
  if (!op) {
    fprintf(stderr, "failed to alloc for op\n");
    goto cleanup;
  }

  // 4. configure op
  rte_crypto_op_attach_sym_session(op, sess);
  op->status = RTE_CRYPTO_OP_STATUS_NOT_PROCESSED;
  op->sym->m_src = m;
  op->sym->m_dst = NULL;
  op->sym->cipher.data.offset = 0;
  op->sym->cipher.data.length = total_len;

  // 5. append IV after op->sym
  uint8_t *iv_ptr = rte_crypto_op_ctod_offset(
      op, uint8_t *,
      sizeof(struct rte_crypto_op) + sizeof(struct rte_crypto_sym_op));
  memcpy(iv_ptr, iv_data, IV_SIZE);

  // 6. enqueue
  struct rte_crypto_op *ops_enq[] = {op};
  int enqueued = rte_cryptodev_enqueue_burst(cdev_id, 0, ops_enq, 1);
  printf("Enqueued %d ops\n", enqueued);
  printf("original op addr: %p\n", op);

  // 7. dequeue
  struct rte_crypto_op *ops_deq[2] = {0};
  int dequeued = 0;
  unsigned int retries = 0;

  while (dequeued == 0 && retries < 100) {
    dequeued = rte_cryptodev_dequeue_burst(cdev_id, 0, ops_deq, 1);
    retries++;
  }

  if (dequeued != 1) {
    fprintf(stderr, "failed to dequeue crypto op, dequeued :%d\n", dequeued);
    goto cleanup;
  }

  // 8. check result
  struct rte_crypto_op *completed_op = ops_deq[0];
  printf("completed_op addr: %p\n", completed_op);
  uint8_t *encrypted_ptr = NULL;
  if (completed_op->status == RTE_CRYPTO_OP_STATUS_SUCCESS) {
    encrypted_ptr = rte_pktmbuf_mtod(completed_op->sym->m_src, uint8_t *);
    printf("\n encryption success\n");
    printf("encrypted data (hex): ");
    for (size_t i = 0; i < total_len; i++) {
      printf("%02x ", encrypted_ptr[i]);
    }
    printf("\n");
  } else {
    fprintf(stderr, "encryption failed with error: %s\n",
            rte_crypto_op_err_msg(completed_op));
    goto cleanup;
  }

  // 9. copy result to return
  *encrypted_text = (char *)malloc(total_len);
  if (!encrypted_text) {
    fprintf(stderr, "failed to alloc for encrypted!\n");
    goto cleanup;
  }
  assert(encrypted_ptr);
  memcpy(*encrypted_text, encrypted_ptr, total_len);
  *encrypted_len = total_len;

  success = true;
  // 10. free resource

cleanup:
  if (op)
    rte_crypto_op_free(op);
  if (m)
    rte_pktmbuf_free(m);
  return success;
}

const char *do_decrypt(uint8_t cdev_id, struct rte_crypto_sym_session *sess,
                       const char *encrypted_text, size_t total_len) {
  char *plaintext = NULL;
  struct rte_crypto_op *op = NULL;
  struct rte_mbuf *m = NULL;

  // 1. alloc mbuf
  m = rte_pktmbuf_alloc(mbuf_pool);
  if (!m) {
    fprintf(stderr, "failed to alloc for mbuf\n");
    goto cleanup;
  }

  // 2. fill in data
  if (rte_pktmbuf_append(m, total_len) == 0) {
    fprintf(stderr, "mbuf append failed\n");
    goto cleanup;
  }
  uint8_t *data_ptr = rte_pktmbuf_mtod(m, uint8_t *);
  memcpy(data_ptr, encrypted_text, total_len);

  // 3. alloc crypto op
  op = rte_crypto_op_alloc(crypto_op_pool, RTE_CRYPTO_OP_TYPE_SYMMETRIC);
  if (!op) {
    fprintf(stderr, "failed to alloc for op\n");
    goto cleanup;
  }

  // 4. configure op
  rte_crypto_op_attach_sym_session(op, sess);
  op->status = RTE_CRYPTO_OP_STATUS_NOT_PROCESSED;
  op->sym->m_src = m;
  op->sym->m_dst = NULL;
  op->sym->cipher.data.offset = 0;
  op->sym->cipher.data.length = total_len;

  // 5. append iv after op->sym
  uint8_t *iv_ptr = rte_crypto_op_ctod_offset(
      op, uint8_t *,
      sizeof(struct rte_crypto_op) + sizeof(struct rte_crypto_sym_op));
  memcpy(iv_ptr, iv_data, IV_SIZE);

  // 6. enqueue
  struct rte_crypto_op *ops_enq[] = {op};
  int enqueued = rte_cryptodev_enqueue_burst(cdev_id, 0, ops_enq, 1);
  printf("Enqueued %d ops\n", enqueued);
  printf("original op addr: %p\n", op);

  // 7. dequeue
  struct rte_crypto_op *ops_deq[2] = {0};
  int dequeued = 0;
  unsigned int retries = 0;

  while (dequeued == 0 && retries < 100) {
    dequeued = rte_cryptodev_dequeue_burst(cdev_id, 0, ops_deq, 1);
    retries++;
  }

  if (dequeued != 1) {
    fprintf(stderr, "failed to dequeue crypto op, dequeued :%d\n", dequeued);
    goto cleanup;
  }

  // 8. check result
  struct rte_crypto_op *completed_op = ops_deq[0];
  printf("completed_op addr: %p\n", completed_op);
  uint8_t *decrypted_ptr = NULL;
  size_t decrypted_data_len = 0;
  if (completed_op->status == RTE_CRYPTO_OP_STATUS_SUCCESS) {
    decrypted_ptr = rte_pktmbuf_mtod(completed_op->sym->m_src, uint8_t *);
    decrypted_data_len = rte_pktmbuf_data_len(completed_op->sym->m_src);
    if (decrypted_data_len != total_len) {
      fprintf(stderr, "decrypted_data_len != total len\n");
    } else {
      printf("decryption success\n");
    }
    printf("decrypted data (hex): ");
    for (size_t i = 0; i < decrypted_data_len; i++) {
      printf("%02x ", decrypted_ptr[i]);
    }
    printf("\n");
  } else {
    fprintf(stderr, "decryption failed with error: %s\n",
            rte_crypto_op_err_msg(completed_op));
    goto cleanup;
  }

  // 9. copy result to return
  plaintext = (char *)malloc(decrypted_data_len);
  if (!plaintext) {
    fprintf(stderr, "failed to alloc for plaintext\n");
    goto cleanup;
  }
  assert(decrypted_ptr);
  memcpy(plaintext, decrypted_ptr, decrypted_data_len);

cleanup:
  if (m)
    rte_pktmbuf_free(m);
  if (op)
    rte_crypto_op_free(op);
  return plaintext;
}

const char* read_file(const char* file_path, size_t *out_len) {
  FILE* fp = fopen(file_path, "r");
  if (!fp) {
    fprintf(stderr, "failed to open file: %s\n", file_path);
    return NULL;
  }
  if (fseek(fp, 0, SEEK_END) != 0) {
    fprintf(stderr, "failed to seek to file end\n");
    goto cleanup;
  }
  long file_size = ftell(fp);
  if (file_size < 0) {
    fprintf(stderr, "failed to get file_size\n");
    goto cleanup;
  }
  rewind(fp);
  char* buffer = (char*)malloc(file_size + 1);
  if (!buffer) {
    fprintf(stderr, "failed to alloc for buffer\n");
    goto cleanup;
  }
  size_t read_bytes = fread(buffer, 1, file_size, fp);
  if (read_bytes != file_size) {
    fprintf(stderr, "failed to read into buffer, expected: %zu, got: %zu\n", file_size, read_bytes);
    goto cleanup;
  }
  buffer[file_size] = '\0';
  fclose(fp);
  *out_len = file_size;
  return buffer;

cleanup:
  if (fp) fclose(fp);
  return NULL;
}

int main(int argc, char **argv) {
  rte_eal_init(argc, argv);

  // 1. cryptodev setup
  uint8_t cdev_id = setup_cryptodev();
  if (cdev_id == (uint8_t)-1) {
    fprintf(stderr, "Error configure cryptodev\n");
    goto cleanup;
  }

  // 2. mbuf & crypto_op mempool alloc
  mbuf_pool =
      rte_pktmbuf_pool_create(MBUF_POOL_NAME, MBUFS_PER_POOL, 256, 0,
                              RTE_MBUF_DEFAULT_BUF_SIZE, rte_socket_id());
  crypto_op_pool = rte_crypto_op_pool_create(
      CRYPTO_OP_POOL_NAME, RTE_CRYPTO_OP_TYPE_SYMMETRIC, CRYPTO_OPS_PER_POOL, 0,
      0, rte_socket_id());

  size_t priv_size = rte_cryptodev_sym_get_private_session_size(cdev_id);
  sess_pool = rte_cryptodev_sym_session_pool_create(
      SESS_POOL_NAME, SESS_PER_POOL, priv_size, 0, 0, rte_socket_id());
  if (!mbuf_pool) {
    fprintf(stderr, "mbuf_pool alloc failed\n");
    goto cleanup;
  }
  if (!crypto_op_pool) {
    fprintf(stderr, "crypto_op_pool alloc failed\n");
    goto cleanup;
  }
  if (!sess_pool) {
    fprintf(stderr, "sess_pool alloc failed\n");
    goto cleanup;
  }

  // 3. create session
  struct rte_crypto_sym_session *encrypt_sess =
      create_sym_encrypt_session(cdev_id);
  struct rte_crypto_sym_session *decrypt_sess =
      create_sym_decrypt_session(cdev_id);
  if (!encrypt_sess || !decrypt_sess) {
    fprintf(stderr, "failed to create session\n");
    goto cleanup;
  }

  // 4. perform encrypt
  const char *plaintext =
      "This is a secrete message for AES-256-CBC encryption. hello world!";
  char *encrypted_text = NULL;
  size_t encrypted_len = 0;
  printf("---start encrypt---\n");
  do_encrypt(cdev_id, encrypt_sess, plaintext, &encrypted_text, &encrypted_len);
  printf("---encrypt complete---\n");
  printf("\n");
  printf("---start decrypt---\n");
  const char *decrypted_text =
      do_decrypt(cdev_id, decrypt_sess, encrypted_text, encrypted_len);
  printf("---decrypt complete---\n");
  printf("---decrypted data---\n");
  printf("%s\n", decrypted_text);

  printf("cleaning resources ...\n");
  rte_mempool_free(mbuf_pool);
  rte_mempool_free(crypto_op_pool);
  rte_mempool_free(sess_pool);
  if (encrypted_text)
    free(encrypted_text);
  if (decrypted_text)
    free((void *)decrypted_text);

  return 0;

cleanup:
  if (mbuf_pool)
    rte_mempool_free(mbuf_pool);
  if (crypto_op_pool)
    rte_mempool_free(crypto_op_pool);
  if (sess_pool)
    rte_mempool_free(sess_pool);
  rte_eal_cleanup();
  return -1;
}
```

## cmake配置
```cmake
cmake_minimum_required(VERSION 3.20)
set(CMAKE_C_COMPILER clang)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_C_STANDARD 23)
set(CMAKE_C_STANDARD_REUIQRED ON)

project(dpdk_shared_mem_learning LANGUAGES C)

# use pkg-config to look up dpdk
find_package(PkgConfig REQUIRED)
if(PKG_CONFIG_FOUND)
  pkg_check_modules(DPDK "libdpdk")
  if(DPDK_FOUND)
    list(JOIN DPDK_LIBRARY_DIRS ":" DPDK_RPATH)
    message(STATUS "found dpdk via pkg-config")
  else()
    message(WARNING "unable to find dpdk via pkg-config")
  endif()
endif()

add_executable(encrypt encrypt.c)
target_link_libraries(encrypt PRIVATE
  ${DPDK_LIBRARIES}
)
```

## 运行命令
```sh
./build/encrypt -l 6-7 --vdev crypto_openssl
```

## 期待输出
```
...
EAL: Selected IOVA mode 'VA'
CRYPTODEV: Creating cryptodev crypto_openssl
CRYPTODEV: Initialisation parameters - name: crypto_openssl,socket id: 0, max queue pairs: 8
---start encrypt---
Original plaintext (padded): '54 68 69 73 20 69 73 20 61 20 73 65 63 72 65 74 65 20 6d 65 73 73 61 67 65 20 66 6f 72 20 41 45 53 2d 32 35 36 2d 43 42 43 20 65 6e 63 72 79 70 74 69 6f 6e 2e 20 68 65 6c 6c 6f 20 77 6f 72 6c 64 21 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Data length: 66 (Padded: 80)
Enqueued 1 ops
original op addr: 0x1010b6700
completed_op addr: 0x1010b6700

 encryption success
encrypted data (hex): 86 1a 24 21 1a 42 0b c6 12 bc b2 e0 f1 1a 7c d6 91 8a f7 a1 f2 ad 09 2e e4 59 f5 7f a4 aa 64 98 06 cb 71 5d b5 95 3b cc 4e 2b 4c ba dc 64 98 8f 09 d2 71 88 4f 8c 03 ee e7 c6 6f a2 52 3f 59 a2 4a 66 e2 94 0f 96 67 54 a5 2c 95 35 99 53 e3 fe
---encrypt complete---

---start decrypt---
Enqueued 1 ops
original op addr: 0x1010b6800
completed_op addr: 0x1010b6800
decryption success
decrypted data (hex): 54 68 69 73 20 69 73 20 61 20 73 65 63 72 65 74 65 20 6d 65 73 73 61 67 65 20 66 6f 72 20 41 45 53 2d 32 35 36 2d 43 42 43 20 65 6e 63 72 79 70 74 69 6f 6e 2e 20 68 65 6c 6c 6f 20 77 6f 72 6c 64 21 00 00 00 00 00 00 00 00 00 00 00 00 00 00
---decrypt complete---
---decrypted data---
This is a secrete message for AES-256-CBC encryption. hello world!
cleaning resources ...
```