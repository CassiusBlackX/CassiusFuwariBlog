---
title: dpdk-do-crypto
published: 2025-12-05
description: 'a sample code to perform a dpdk sym crypto op'
image: ''
tags: ['dpdk', 'crypto']
category: deployment
draft: false 
lang: ''
---
# dpdk执行对称加解密aes-256-cbc
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

#define MBUF_SIZE 16384

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

  // printf("Original plaintext (padded): '");
  // for (size_t i = 0; i < total_len; i++) {
  //   printf("%02x ", data_ptr[i]);
  // }
  // printf("\n");
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
    // printf("encrypted data (hex): ");
    // for (size_t i = 0; i < total_len; i++) {
    //   printf("%02x ", encrypted_ptr[i]);
    // }
    // printf("\n");
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
    // printf("decrypted data (hex): ");
    // for (size_t i = 0; i < decrypted_data_len; i++) {
    //   printf("%02x ", decrypted_ptr[i]);
    // }
    // printf("\n");
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
                              MBUF_SIZE, rte_socket_id());
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
  // const char *plaintext =
      // "This is a secrete message for AES-256-CBC encryption. hello world!";
  size_t plaintext_len = 0;
  const char* plaintext = read_file(__FILE__, &plaintext_len);
  printf("*** plaintext len: %zu\n", plaintext_len);
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
  // printf("%s\n", decrypted_text);

  printf("cleaning resources ...\n");
  if (encrypted_text)
    free(encrypted_text);
  if (decrypted_text)
    free((void *)decrypted_text);

cleanup:
  if (encrypt_sess) rte_cryptodev_sym_session_free(cdev_id, encrypt_sess);
  if (decrypt_sess) rte_cryptodev_sym_session_free(cdev_id, decrypt_sess);
  if (mbuf_pool)
    rte_mempool_free(mbuf_pool);
  if (crypto_op_pool)
    rte_mempool_free(crypto_op_pool);
  if (sess_pool)
    rte_mempool_free(sess_pool);
  rte_cryptodev_stop(cdev_id);
  rte_eal_cleanup();
  return 0;
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
# 如果是读文件，则不会有下面这行输出，否则太长了
Original plaintext (padded): '54 68 69 73 20 69 73 20 61 20 73 65 63 72 65 74 65 20 6d 65 73 73 61 67 65 20 66 6f 72 20 41 45 53 2d 32 35 36 2d 43 42 43 20 65 6e 63 72 79 70 74 69 6f 6e 2e 20 68 65 6c 6c 6f 20 77 6f 72 6c 64 21 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Data length: 66 (Padded: 80)
Enqueued 1 ops
original op addr: 0x1010b6700
completed_op addr: 0x1010b6700

 encryption success
# 如果是读文件，则不会有下面这行输出，否则太长了
encrypted data (hex): 86 1a 24 21 1a 42 0b c6 12 bc b2 e0 f1 1a 7c d6 91 8a f7 a1 f2 ad 09 2e e4 59 f5 7f a4 aa 64 98 06 cb 71 5d b5 95 3b cc 4e 2b 4c ba dc 64 98 8f 09 d2 71 88 4f 8c 03 ee e7 c6 6f a2 52 3f 59 a2 4a 66 e2 94 0f 96 67 54 a5 2c 95 35 99 53 e3 fe
---encrypt complete---

---start decrypt---
Enqueued 1 ops
original op addr: 0x1010b6800
completed_op addr: 0x1010b6800
decryption success
# 如果是读文件，则不会有下面这行输出，否则太长了
decrypted data (hex): 54 68 69 73 20 69 73 20 61 20 73 65 63 72 65 74 65 20 6d 65 73 73 61 67 65 20 66 6f 72 20 41 45 53 2d 32 35 36 2d 43 42 43 20 65 6e 63 72 79 70 74 69 6f 6e 2e 20 68 65 6c 6c 6f 20 77 6f 72 6c 64 21 00 00 00 00 00 00 00 00 00 00 00 00 00 00
---decrypt complete---
---decrypted data---
This is a secrete message for AES-256-CBC encryption. hello world!
cleaning resources ...
```

# dpdk执行非对称加解密rsa2048
## 代码
```c
#include <stdio.h>
#include <stdbool.h>
#include <assert.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

#include <openssl/pem.h>
#include <openssl/rsa.h>
#include <openssl/bn.h>
#include <openssl/types.h>
#include <openssl/evp.h>
#include <openssl/core_names.h>

#include <rte_eal.h>
#include <rte_malloc.h>
#include <rte_mempool.h>
#include <rte_crypto.h>
#include <rte_cryptodev.h>
#include <rte_crypto_asym.h>
#include <rte_lcore.h>

#define ASYM_SESS_POOL_NAME "asym_sess_pool"
#define ASYM_OP_POOL_NAME "asym_op_pool"
#define ASYM_OPS_PER_POOL 2048
#define MBUFS_PER_POOL 2048
#define ASYM_SESS_PER_POOL 2048
#define PMD_NAME "crypto_openssl"
#define RSA_KEY_SIZE_BITS 2048
#define RSA_KEY_SIZE_BYTES (RSA_KEY_SIZE_BITS / 8)

#define PUBLIC_KEY_FILE "public.pem"
#define PRIVATE_KEY_FILE "private.pem"

static struct rte_mempool *asym_op_pool = NULL;
static struct rte_mempool *asym_sess_pool = NULL;

typedef struct {
  // modules
  uint8_t n[RSA_KEY_SIZE_BYTES];
  // public exponent
  uint8_t e[RSA_KEY_SIZE_BYTES];
  // private exponent
  uint8_t d[RSA_KEY_SIZE_BYTES];
  // exact len, since `E` might be smaller than 256
  size_t e_len;
} RsaKeyData;

bool load_rsa_key_components(RsaKeyData* data) {
    bool ret = false;
    
    FILE* fp = fopen(PRIVATE_KEY_FILE, "r");
    if (!fp) {
        fprintf(stderr, "Error: failed to open rsa private key file\n");
        goto cleanup;
    }

    // 1. read private key 
    EVP_PKEY *pkey = PEM_read_PrivateKey(fp, NULL, NULL, NULL);
    fclose(fp);
    fp = NULL;
    if (!pkey) {
        fprintf(stderr, "failed to read RSA private key from PEM\n");
        goto cleanup;
    }

    // 2. check if is RSA!
    if (EVP_PKEY_id(pkey) != EVP_PKEY_RSA) {
        fprintf(stderr, "EVP_PKEY is not an RSA key\n");
        goto cleanup;
    }

    // 3. extract private key infos
    BIGNUM *n = NULL, *e = NULL, *d = NULL;
    // OSSL_PKEY_PARAM_RSA_N: modulus
    if (EVP_PKEY_get_bn_param(pkey, OSSL_PKEY_PARAM_RSA_N, &n) != 1) {
        fprintf(stderr, "failed to extract modulus N\n");
        goto cleanup;
    }
    // OSSL_PKEY_PARAM_RSA_E: public exponent
    if (EVP_PKEY_get_bn_param(pkey, OSSL_PKEY_PARAM_RSA_E, &e) != 1) {
        fprintf(stderr, "failed to extract public exponent E\n");
        goto cleanup;
    }
    // OSSL_PKEY_PARAM_RSA_D: private exponent
    if (EVP_PKEY_get_bn_param(pkey, OSSL_PKEY_PARAM_RSA_D, &d) != 1) {
        fprintf(stderr, "failed to extract private exponent D\n");
        goto cleanup;
    }
    
    // 4.transform BIGNUM into Big Endian mode
    
    // Modulus (N)
    if (BN_bn2binpad(n, data->n, RSA_KEY_SIZE_BYTES) != RSA_KEY_SIZE_BYTES) {
        fprintf(stderr, "failed to convert N to binary\n");
        goto cleanup;
    }

    // Public Exponent (E)
    data->e_len = BN_bn2binpad(e, data->e, RSA_KEY_SIZE_BYTES);
    if (data->e_len <= 0) {
        fprintf(stderr, "failed to convert E to binary\n");
        goto cleanup;
    }

    // Private Exponent (D)
    if (BN_bn2binpad(d, data->d, RSA_KEY_SIZE_BYTES) != RSA_KEY_SIZE_BYTES) {
        fprintf(stderr, "failed to convert D to binary\n");
        goto cleanup;
    }
    
    ret = true;

cleanup:
    if (fp) fclose(fp);
    // 释放 BIGNUM 结构 (它们在 EVP_PKEY_get_bn_param 中被分配)
    if (n) BN_free(n);
    if (e) BN_free(e);
    if (d) BN_free(d);
    // 释放 EVP_PKEY
    if (pkey) EVP_PKEY_free(pkey);
    return ret;
}

static const char* rte_crypto_op_err_msg(const struct rte_crypto_op *op) {
  switch (op->status) {
    case RTE_CRYPTO_OP_STATUS_SUCCESS:
      return "success";
    case RTE_CRYPTO_OP_STATUS_NOT_PROCESSED:
      return "Not Processed";
    case RTE_CRYPTO_OP_STATUS_AUTH_FAILED:
      return "auth failed";
    case RTE_CRYPTO_OP_STATUS_INVALID_SESSION:
      return "invalid session";
    case RTE_CRYPTO_OP_STATUS_INVALID_ARGS:
      return "invalid args";
    case RTE_CRYPTO_OP_STATUS_ERROR:
      return "error";
    default:
      return "unknown error";
  }
}

static int setup_cryptodev() {
  uint8_t cdev_ids[64] = {0};
  int num_cdevs = rte_cryptodev_devices_get(PMD_NAME, cdev_ids, 64);
  assert(num_cdevs > 0 && "no cdev" PMD_NAME "found");
  uint8_t cdev_id = cdev_ids[0];

  struct rte_cryptodev_config config = {
    .nb_queue_pairs = 1,
    .socket_id = rte_socket_id(),
  };
  if (rte_cryptodev_configure(cdev_id, &config) < 0) {
    fprintf(stderr, "failed to configure crypto dev: %u\n", cdev_id);
    return -1;
  }

  struct rte_cryptodev_qp_conf qp_conf = {
    .nb_descriptors = 2048,
  };
  if (rte_cryptodev_queue_pair_setup(cdev_id, 0, &qp_conf, rte_socket_id()) < 0) {
    fprintf(stderr, "failed to setup queue pair 0 on crypto dev: %u\n", cdev_id);
    return -1;
  }

  if (rte_cryptodev_start(cdev_id) < 0) {
    fprintf(stderr, "failed to start crypto dev: %u\n", cdev_id);
    return -1;
  }

  return cdev_id;
}

static struct rte_cryptodev_asym_session *create_asym_session(uint8_t cdev_id, const RsaKeyData *key_data) {
  // xform for private key
  struct rte_crypto_rsa_xform rsa_xform = {
    .key_type = RTE_RSA_KEY_TYPE_EXP,
    .n = {.data = (uint8_t *)key_data->n, .length = RSA_KEY_SIZE_BYTES},
    .e = {.data = (uint8_t *)key_data->e, .length = RSA_KEY_SIZE_BYTES},
    .d = {.data = (uint8_t *)key_data->d, .length = RSA_KEY_SIZE_BYTES},
    .padding.type = RTE_CRYPTO_RSA_PADDING_PKCS1_5,
  };
  struct rte_crypto_asym_xform asym_xform = {
    .next = NULL,
    .xform_type = RTE_CRYPTO_ASYM_XFORM_RSA,
    .rsa = rsa_xform,
  };
  struct rte_cryptodev_asym_session *sess = NULL;
  int ret = rte_cryptodev_asym_session_create(cdev_id, &asym_xform, asym_sess_pool, (void**)&sess);
  if (ret != 0 || !sess) {
    fprintf(stderr, "failed to create asym_session\n");
    return NULL;
  }
  return sess;
}

static bool do_asym_op(uint8_t cdev_id, struct rte_crypto_op *op) {
  struct rte_crypto_op *ops_enq[] = {op};
  struct rte_crypto_op *ops_deq[1] = {NULL};
  int enqueued = 0, dequeued = 0;
  unsigned int retires = 0;

  // 1. enqueue
  enqueued = rte_cryptodev_enqueue_burst(cdev_id, 0, ops_enq, 1);
  if (enqueued != 1) {
    fprintf(stderr, "failed to enqueue op, retrying later\n");
    return false;
  }

  // 2. dequeue (poll for completion)
  while (dequeued == 0 && retires < 1000) {
    dequeued = rte_cryptodev_dequeue_burst(cdev_id, 0, ops_deq, 1);
    retires++;
  }
  if (dequeued != 1) {
    fprintf(stderr, "failed to dequeue crypto op\n");
    return false;
  }

  // 3. check result
  struct rte_crypto_op *completed_op = (struct rte_crypto_op*)ops_deq[0];
  if (completed_op->status != RTE_CRYPTO_OP_STATUS_SUCCESS) {
    fprintf(stderr, "operation failed, status: %s\n", rte_crypto_op_err_msg(completed_op));
    return false;
  }

  return true;
}

bool perform_rsa_test(uint8_t cdev_id, const RsaKeyData *key_data, struct rte_cryptodev_asym_session* sess) {
  bool ret = false;

  // 1. data prepare
  const char* plaintext = "DPDK rsa2048 test message";
  size_t plaintext_len = strlen(plaintext) + 1;

  uint8_t *in_buf = rte_zmalloc("in_buf", RSA_KEY_SIZE_BYTES, 0);
  if (!in_buf) {
    fprintf(stderr, "failed to alloc for in_buf\n");
    goto cleanup;
  }
  uint8_t *cipher_buf = rte_zmalloc("cipher_buf", RSA_KEY_SIZE_BYTES, 0);
  if (!cipher_buf) {
    fprintf(stderr, "failed to alloc for cipher_buf\n");
    goto cleanup;
  }
  uint8_t *decrypt_buf = rte_zmalloc("decrypt_buf", RSA_KEY_SIZE_BYTES, 0);
  if (!decrypt_buf) {
    fprintf(stderr, "failed to alloc for decrypt buf\n");
    goto cleanup;
  }

  memcpy(in_buf, plaintext, plaintext_len > RSA_KEY_SIZE_BYTES ? RSA_KEY_SIZE_BYTES : plaintext_len);
  printf("original data (%zu bytes): %s\n", plaintext_len, plaintext);

  // 3. alloc crypto op
  struct rte_crypto_op *op_enc = rte_crypto_op_alloc(asym_op_pool, RTE_CRYPTO_OP_TYPE_ASYMMETRIC);
  if (!op_enc) {
    fprintf(stderr, "failed to alloc for op_enc\n");
    goto cleanup;
  }
  struct rte_crypto_op *op_dec = rte_crypto_op_alloc(asym_op_pool, RTE_CRYPTO_OP_TYPE_ASYMMETRIC);
  if (!op_dec) {
    fprintf(stderr, "failed to alloc for op_dec\n");
    goto cleanup;
  }

  // 4. fill op_enc
  rte_crypto_op_attach_asym_session(op_enc, sess);
  op_enc->asym->rsa.op_type = RTE_CRYPTO_ASYM_OP_ENCRYPT;
  op_enc->asym->rsa.message.data = in_buf;
  op_enc->asym->rsa.message.length = plaintext_len;
  op_enc->asym->rsa.cipher.data = cipher_buf;
  op_enc->asym->rsa.cipher.length = RSA_KEY_SIZE_BYTES;

  // 5. do asym op encrypt
  if (!do_asym_op(cdev_id, op_enc)) {
    fprintf(stderr, "RSA encryption failed\n");
    goto cleanup;    
  }
  printf("Encryption successful, ciphertext length: %zu bytes\n", op_enc->asym->rsa.cipher.length);
  
  // 6. do asym op decrypt
  rte_crypto_op_attach_asym_session(op_dec, sess);
  op_dec->asym->rsa.op_type = RTE_CRYPTO_ASYM_OP_DECRYPT;
  op_dec->asym->rsa.cipher.data = cipher_buf;
  op_dec->asym->rsa.cipher.length = RSA_KEY_SIZE_BYTES;
  op_dec->asym->rsa.message.data = decrypt_buf;
  op_dec->asym->rsa.message.length = RSA_KEY_SIZE_BYTES;

  if (!do_asym_op(cdev_id, op_dec)) {
    fprintf(stderr, "RSA decryption failed\n");
    goto cleanup;
  }
  size_t decrypted_len = op_dec->asym->rsa.message.length;
  printf("Decryption successful, plaintext length: %zu bytes\n", decrypted_len);

  // 7. check result
  printf("Original message:\n%s\n", plaintext);
  printf("Decrypted message:\n");
  char* decrypted_text = (char*)malloc(decrypted_len + 1);
  memcpy(decrypted_text, op_dec->asym->rsa.message.data, decrypted_len);
  decrypted_text[decrypted_len] = '\0';
  printf("%s\n", decrypted_text);
  ret = true;

cleanup:
  if (in_buf) rte_free(in_buf);
  if (cipher_buf) rte_free(cipher_buf);
  if (decrypt_buf) rte_free(decrypt_buf);
  if (op_enc) rte_crypto_op_free(op_enc);
  if (op_dec) rte_crypto_op_free(op_dec);
  return ret;
}

int main(int argc, char**argv) {
  // 1. read rsa data
  RsaKeyData *key_data = malloc(sizeof(RsaKeyData));
  if (!load_rsa_key_components(key_data)) {
    fprintf(stderr, "failed to load rsa data\n");
    goto cleanup;
  }
  
  rte_eal_init(argc, argv);

  // 2. cryptodev setup
  uint8_t cdev_id = setup_cryptodev();
  if (cdev_id == (uint8_t)-1) {
    fprintf(stderr, "Error configure cryptodev\n");
    goto cleanup;
  }

  // 3. sess & crypto_op mempool alloc
  asym_op_pool = rte_crypto_op_pool_create(
      ASYM_OP_POOL_NAME, RTE_CRYPTO_OP_TYPE_ASYMMETRIC, ASYM_OPS_PER_POOL, 0,
      sizeof(struct rte_crypto_asym_op), rte_socket_id());
  size_t priv_size = rte_cryptodev_asym_get_private_session_size(cdev_id);
  asym_sess_pool = rte_cryptodev_asym_session_pool_create(ASYM_SESS_POOL_NAME, ASYM_SESS_PER_POOL, 0, priv_size, rte_socket_id());
  if (!asym_op_pool) {
    fprintf(stderr, "asym_op_pool alloc failed\n");
    goto cleanup;
  }
  if (!asym_sess_pool) {
    fprintf(stderr, "failed to alloc asym_sess_pool\n");
    goto cleanup;
  }

  // 4. create session
  struct rte_cryptodev_asym_session *sess = create_asym_session(cdev_id, key_data);

  // 5. perform test
  perform_rsa_test(cdev_id, key_data, sess);

  printf("test complete, cleaning resource...\n");
cleanup:
  if (sess) rte_cryptodev_asym_session_free(cdev_id, sess);
  if (asym_op_pool) rte_mempool_free(asym_op_pool);
  if (asym_sess_pool) rte_mempool_free(asym_sess_pool);
  rte_cryptodev_stop(cdev_id);
  rte_eal_cleanup();
  return 0;
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

add_executable(rsa_enc_dec rsa_enc_dec.c)
target_link_libraries(rsa_enc_dec PRIVATE
  ${DPDK_LIBRARIES}
  crypto
)
```

## 生成公私钥
生成私钥
```sh
openssl genpkey -algorithm RSA -out private.pem -pkeyopt rsa_keygen_bits:2048
```
生成公钥(可选)，因为公钥是可以从private.pem中获取相关信息的
```sh
openssl rsa -in private.pem -pubout -out public.pem
```

## 运行命令
```sh
./build/rsa_enc_dec -l 6-7 --vdev crypto_openssl
```

## 期待输出
```
EAL: Detected CPU lcores: 320
EAL: Detected NUMA nodes: 4
EAL: Detected shared linkage of DPDK
EAL: Multi-process socket /tmp/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'VA'
CRYPTODEV: Creating cryptodev crypto_openssl
CRYPTODEV: Initialisation parameters - name: crypto_openssl,socket id: 0, max queue pairs: 8
original data (26 bytes): DPDK rsa2048 test message
Encryption successful, ciphertext length: 256 bytes
Decryption successful, plaintext length: 26 bytes
Original message:
DPDK rsa2048 test message
Decrypted message:
DPDK rsa2048 test message
test complete, cleaning resource...
CRYPTODEV: Closing crypto device crypto_openssl
```

# dpdk使用rsa进行sign和verify
## 代码
包括main函数在内的其它所有函数同上，只有原来的`perform_rsa_test`现在变成了下面的这个`perform_rsa_sign_verify_test`
```c
static bool perform_rsa_sign_verify_test(uint8_t cdev_id, const RsaKeyData *key_data, struct rte_cryptodev_asym_session *sess) {
  // 1. param check
  if (!key_data || !sess) {
    fprintf(stderr, "nullptr input\n");
    return false;
  }
  bool ret = false;

  // 2. prepare plaintext
  const char *plaintext = "DPDK rsa sign test message";
  size_t plaintext_len = strlen(plaintext);

  uint8_t *plain_buf = rte_zmalloc("msg_buf", RSA_KEY_SIZE_BYTES, 0);
  uint8_t *sign_buf = rte_zmalloc("sign_buf", RSA_KEY_SIZE_BYTES, 0);
  if (!plain_buf || !sign_buf) {
    fprintf(stderr, "failed to alloc for buf\n");
    return false;
  }
  memcpy(plain_buf, plaintext, plaintext_len);

  // 3. alloc crypto op
  struct rte_crypto_op *op = rte_crypto_op_alloc(asym_op_pool, RTE_CRYPTO_OP_TYPE_ASYMMETRIC);
  if (!op) {
    fprintf(stderr, "failed to alloc for crypto op");
    goto cleanup;
  }

  // 4. fill in op for sign
  rte_crypto_op_attach_asym_session(op, sess);
  op->status = RTE_CRYPTO_OP_STATUS_NOT_PROCESSED;
  op->asym->rsa.op_type = RTE_CRYPTO_ASYM_OP_SIGN;
  op->asym->rsa.message.data = plain_buf;
  op->asym->rsa.message.length = plaintext_len;
  op->asym->rsa.sign.data = sign_buf;
  op->asym->rsa.sign.length = RSA_KEY_SIZE_BYTES;

  // 5. sign
  if (!do_asym_op(cdev_id, op)) {
    fprintf(stderr, "rsa sign failed\n");
    goto cleanup;
  }
  printf("RSA sign success, signature generated\n");

  // 6. fill in op for verify
  op->status = RTE_CRYPTO_OP_STATUS_NOT_PROCESSED;
  op->asym->rsa.op_type = RTE_CRYPTO_ASYM_OP_VERIFY;
  op->asym->rsa.sign.data = sign_buf;
  op->asym->rsa.sign.length = RSA_KEY_SIZE_BYTES;
  op->asym->rsa.message.data = plain_buf;
  op->asym->rsa.message.length = plaintext_len;
  
  // 7. verify
  if (!do_asym_op(cdev_id, op)) {
    fprintf(stderr, "RSA verify failed\n");
    goto cleanup;
  } else {
    printf("verify success\n");
    ret = true;
  }

  // 8. mallicious verify
  // signed data bytes reverted
  uint8_t *err_buf = rte_zmalloc("err_buf", RSA_KEY_SIZE_BYTES, 0);
  memcpy(err_buf, sign_buf, RSA_KEY_SIZE_BYTES);
  err_buf[0] ^= 0xff;
  op->status = RTE_CRYPTO_OP_STATUS_NOT_PROCESSED;
  op->asym->rsa.op_type = RTE_CRYPTO_ASYM_OP_VERIFY;
  op->asym->rsa.sign.data = err_buf;
  op->asym->rsa.sign.length = RSA_KEY_SIZE_BYTES;
  op->asym->rsa.message.data = plain_buf;
  op->asym->rsa.message.length = plaintext_len;
  bool err_res = do_asym_op(cdev_id, op);
  if (!err_res) {
    printf("mallicious verify FAILED as expected, status: %s\n", rte_crypto_op_err_msg(op));
  } else {
    fprintf(stderr, "mallicious check failed\n");
    ret = false;
    goto cleanup;
  }

cleanup:
  if (plain_buf) rte_free(plain_buf);
  if (sign_buf) rte_free(sign_buf);
  if (err_buf) rte_free(err_buf);
  if (op) rte_crypto_op_free(op);
  return ret;
}
```

## 期待输出
```
EAL: Selected IOVA mode 'VA'
CRYPTODEV: Creating cryptodev crypto_openssl
CRYPTODEV: Initialisation parameters - name: crypto_openssl,socket id: 0, max queue pairs: 8
RSA sign success, signature generated
verify success
operation failed, status: error
mallicious verify FAILED as expected, status: error
test complete, cleaning resource...
CRYPTODEV: Closing crypto device crypto_openssl
```