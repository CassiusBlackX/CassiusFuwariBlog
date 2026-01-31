---
title: af_alg-crypto
published: 2026-02-01
description: '调用AF_ALG API执行加解密'
image: ''
tags: []
category: ''
draft: false 
lang: ''
---
# 通过调用linux kernel的AF_ALG API执行加解密
```c
#define _GNU_SOURCE
#include <linux/if_alg.h>
#include <stddef.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <unistd.h>

#define AES_BLOCK_SIZE 16
#define KEY_SIZE 32 /* AES-256 */
#define IV_SIZE 16

/* PKCS#7 padding */
static unsigned char *pkcs7_pad(const unsigned char *in, size_t inlen,
                                size_t *outlen) {
  size_t pad = AES_BLOCK_SIZE - (inlen % AES_BLOCK_SIZE);
  *outlen = inlen + pad;

  unsigned char *out = malloc(*outlen);
  if (!out)
    return NULL;

  memcpy(out, in, inlen);
  memset(out + inlen, pad, pad);

  return out;
}

static int pkcs7_unpad(unsigned char *buf, size_t *len) {
  if (*len == 0)
    return -1;

  unsigned char pad = buf[*len - 1];
  if (pad == 0 || pad > AES_BLOCK_SIZE)
    return -1;

  for (size_t i = 0; i < pad; i++) {
    if (buf[*len - 1 - i] != pad)
      return -1;
  }

  *len -= pad;
  return 0;
}

/* AF_ALG encrypt/decrypt */
static int afalg_crypt(int enc, /* 1 = encrypt, 0 = decrypt */
                       const unsigned char *key, const unsigned char *iv,
                       const unsigned char *in, size_t inlen,
                       unsigned char *out) {
  struct sockaddr_alg sa = {
      .salg_family = AF_ALG,
      .salg_type = "skcipher",
      .salg_name = "cbc(aes)",
  };

  int tfmfd = socket(AF_ALG, SOCK_SEQPACKET, 0);
  if (tfmfd == -1) {
    perror("socket");
    return -1;
  }

  if (bind(tfmfd, (struct sockaddr *)&sa, sizeof(sa)) != 0) {
    perror("bind");
    close(tfmfd);
    return -1;
  }

  if (setsockopt(tfmfd, SOL_ALG, ALG_SET_KEY, key, KEY_SIZE) != 0) {
    perror("setsockopt(key)");
    close(tfmfd);
    return -1;
  }

  int opfd = accept(tfmfd, NULL, 0);
  if (opfd == -1) {
    perror("accept");
    close(tfmfd);
    return -1;
  }

  struct msghdr msg = {0};
  struct cmsghdr *cmsg;
  char cbuf[CMSG_SPACE(sizeof(struct af_alg_iv) + IV_SIZE) +
            CMSG_SPACE(sizeof(uint32_t))];
  struct iovec iov = {
      .iov_base = (void *)in,
      .iov_len = inlen,
  };

  msg.msg_iov = &iov;
  msg.msg_iovlen = 1;
  msg.msg_control = cbuf;
  msg.msg_controllen = sizeof(cbuf);

  cmsg = CMSG_FIRSTHDR(&msg);
  cmsg->cmsg_level = SOL_ALG;
  cmsg->cmsg_type = ALG_SET_OP;
  cmsg->cmsg_len = CMSG_LEN(sizeof(uint32_t));
  *(uint32_t *)CMSG_DATA(cmsg) = enc ? ALG_OP_ENCRYPT : ALG_OP_DECRYPT;

  cmsg = CMSG_NXTHDR(&msg, cmsg);
  cmsg->cmsg_level = SOL_ALG;
  cmsg->cmsg_type = ALG_SET_IV;
  cmsg->cmsg_len = CMSG_LEN(sizeof(struct af_alg_iv) + IV_SIZE);
  struct af_alg_iv *aiv = (struct af_alg_iv*)CMSG_DATA(cmsg);
  aiv->ivlen = IV_SIZE;
  memcpy(aiv->iv, iv, IV_SIZE);

  if (sendmsg(opfd, &msg, 0) < 0) {
    perror("sendmsg");
    close(opfd);
    close(tfmfd);
    return -1;
  }

  if (read(opfd, out, inlen) != (ssize_t)inlen) {
    perror("read");
    close(opfd);
    close(tfmfd);
    return -1;
  }

  close(opfd);
  close(tfmfd);
  return 0;
}

int main(void) {
  const unsigned char key[KEY_SIZE] = {
      0x60, 0x3d, 0xeb, 0x10, 0x15, 0xca, 0x71, 0xbe, 0x2b, 0x73, 0xae,
      0xf0, 0x85, 0x7d, 0x77, 0x81, 0x1f, 0x35, 0x2c, 0x07, 0x3b, 0x61,
      0x08, 0xd7, 0x2d, 0x98, 0x10, 0xa3, 0x09, 0x14, 0xdf, 0xf4};

  const unsigned char iv[IV_SIZE] = {0x00, 0x01, 0x02, 0x03, 0x04, 0x05,
                                     0x06, 0x07, 0x08, 0x09, 0x0a, 0x0b,
                                     0x0c, 0x0d, 0x0e, 0x0f};

  const unsigned char plaintext[] =
      "Hello AF_ALG AES-256-CBC with PKCS#7 padding!";
  size_t pt_len = strlen((char *)plaintext);

  /* pad */
  size_t padded_len;
  unsigned char *padded = pkcs7_pad(plaintext, pt_len, &padded_len);
  if (!padded) {
    fprintf(stderr, "padding failed\n");
    return 1;
  }

  unsigned char *cipher = malloc(padded_len);
  unsigned char *decrypted = malloc(padded_len);

  if (!cipher || !decrypted) {
    fprintf(stderr, "malloc failed\n");
    return 1;
  }

  /* encrypt */
  if (afalg_crypt(1, key, iv, padded, padded_len, cipher) != 0) {
    fprintf(stderr, "encrypt failed\n");
    return 1;
  }

  /* decrypt */
  if (afalg_crypt(0, key, iv, cipher, padded_len, decrypted) != 0) {
    fprintf(stderr, "decrypt failed\n");
    return 1;
  }

  size_t dec_len = padded_len;
  if (pkcs7_unpad(decrypted, &dec_len) != 0) {
    fprintf(stderr, "unpad failed\n");
    return 1;
  }

  /* compare */
  if (dec_len != pt_len || memcmp(decrypted, plaintext, pt_len) != 0) {
    fprintf(stderr, "Failure! decrypt result mismatch\n");
    free(padded);
    free(cipher);
    free(decrypted);
    return 1;
  }

  printf("Success! AES-256-CBC encrypt/decrypt OK\n");
  printf("Plaintext : %s\n", plaintext);
  printf("Decrypted : %.*s\n", (int)dec_len, decrypted);

  free(padded);
  free(cipher);
  free(decrypted);
  return 0;
}
```