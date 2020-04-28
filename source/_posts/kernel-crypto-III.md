---
title: kernel_crypto_III
date: 2019-08-07 14:46:14
tags: cryptographic
categories: drivers
---

### 1. Method with cryptographic
用户空间的方式有如下方式：
- netlink (AF_ALG)  socket 的形式（kernel 已经merge进入主线） 
- ioctl （OpenBSD 的形式， kenrel 没有merge此部分代码  ）
- openssl

我们常见的userspace 与kernel space 之间通信的方式有：
- IOCTL
- /proc
- /sys
- netlink

我们主张使用socket 的方式，更为优雅与灵活。有人在此基础上封装了lib， [libkcapi](http://www.chronox.de/libkcapi.html)

<!--more-->

### 2. netlink(socket)
#### 2.1. data structure
<linux/if_alg.h>
```c
#include <linux/types.h>
struct sockaddr_alg {
        __u16   salg_family;
        __u8    salg_type[14];
        __u32   salg_feat;
        __u32   salg_mask;
        __u8    salg_name[64];
};
struct af_alg_iv {
        __u32   ivlen;
        __u8    iv[0];
};
/* Socket options */
#define ALG_SET_KEY                     1
#define ALG_SET_IV                      2
#define ALG_SET_OP                      3

/* Operations */
#define ALG_OP_DECRYPT                  0
#define ALG_OP_ENCRYPT                  1
```

Currently, the following ciphers are accessible:
- Message digest including keyed message digest (HMAC, CMAC)  
- Symmetric ciphers  
- AEAD ciphers  
- Random Number Generators  

我们可以使用如下的sockaddr_alg 的实例联系到具体的算法上。

```c
/* salg_type support:
    - hash
    - skcipher
    - ahead
    - rnd
*/

struct sockaddr_alg sa = {
    .salg_family = AF_ALG,
    .salg_type = "skcipher", /* this selects the symmetric cipher */
    .salg_name = "cbc(aes)" /* this is the cipher name */
};
```

socket 中的family AF_ALG, setsockopt 使用 SOL_ALG。如果header 中没有申明，可以使用如下定义：  
```c
#ifndef AF_ALG
#define AF_ALG 38
#endif
#ifndef SOL_ALG
#define SOL_ALG 279
#endif
```
#### 2.2. Kernel internal 
在kenrel 内部注册了AF_ALG family 类型的socket。在kernel/crypto/af_alg.c

```c
static struct proto alg_proto = {
	.name			= "ALG",
	.memory_allocated	= &alg_memory_allocated,
};

#define PF_ALG		AF_ALG

static const struct net_proto_family alg_family = {
	.family	=	PF_ALG,
	.create	=	alg_create,
};

static int __init af_alg_init(void)
{
	int err = proto_register(&alg_proto, 0);

	err = sock_register(&alg_family);
	
	return err;
}
```

在kernel/crypto/algif_skcipher.c， 类似的algif_hash.c 注册了hash 类型的对象。
```c
static struct proto_ops algif_skcipher_ops = {
	.family		=	PF_ALG,

	.release	=	af_alg_release,
	.sendmsg	=	skcipher_sendmsg,
	.sendpage	=	skcipher_sendpage,
	.recvmsg	=	skcipher_recvmsg,
	.poll		=	skcipher_poll,
};

static const struct af_alg_type algif_type_skcipher = {
	.bind		=	skcipher_bind,
	.release	=	skcipher_release,
	.setkey		=	skcipher_setkey,
	.accept		=	skcipher_accept_parent,
	.ops		=	&algif_skcipher_ops,
	.name		=	"skcipher",
};

static int __init algif_skcipher_init(void)
{
	return af_alg_register_type(&algif_type_skcipher);
}
```

![kernel af_alg system call image](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/af_alg%20kernel%20flow.png)

#### 2.3. usage flow
我们使用如下流程进行使用：
1. create socket with AF_ALG  
2. bind socket with sockaddr_alg addr. 
3. set key, setsocketopt()
4. accept with socket. accept system call return new file descriptor.
5. use new fd to sendmsg(), recvmsg()

__Setsockopt Interface__
我们可以是用setsockopt（） 系统调用进行设定。

```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int getsockopt(int sockfd, int level, int optname,
                void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname,
                const void *optval, socklen_t optlen);
```

#### 2.4. usage
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <sys/socket.h>
#include <linux/if_alg.h>

#ifndef AF_ALG
#define AF_ALG 38
#define SOL_ALG 279
#endif

extern errno;

#define PRINT_ROW_LEN 15
void print_hex_dump(char *src, int n)
{
    int i;
    for(i=0; i<n; i++) {
        printf("%02x ", src[i]);
      if (i % PRINT_ROW_LEN == 0 && i != 0)
        printf("\n");
    }
    printf("\n");
}

int setkey(int sockfd, char * key, int keylen)
{
    int err = setsockopt(sockfd, SOL_ALG, ALG_SET_KEY, key, keylen);
    
    if (err) {
      perror("setsockopt err");
      goto out;
    }
    printf("setkey success\n");
out:
    err = errno;
    return err;    
}

int sendmsg_to_cipher(int sockfd, int cmsg_type, int operation, char * src, int len)
{
    int err = 0;
    struct msghdr msg;
    struct iovec iov;
    struct cmsghdr* cmsg = malloc(CMSG_SPACE(sizeof(operation)));
    if (cmsg == NULL) {
        perror("malloc setop_cmsg err");
        goto out;
    }
    cmsg->cmsg_len = CMSG_SPACE(sizeof(operation));
    cmsg->cmsg_level = SOL_ALG;
    cmsg->cmsg_type = cmsg_type;
    memcpy(CMSG_DATA(cmsg), &operation, sizeof(operation));

    iov.iov_base = src;
    iov.iov_len = len;

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    msg.msg_control = cmsg;
    msg.msg_controllen = CMSG_SPACE(sizeof(unsigned int));
    msg.msg_flags = 0;
    err = sendmsg(sockfd, &msg, 0);
    if (err == -1) {
        perror("sendmsg operation err");
        goto send_msg_err;
    }
    printf("sendmsg success\n");

send_msg_err:
    free(cmsg);
out:
    err = errno;
    return err;
}

int recvmsg_from_cipher(int sockfd, char *src, int len)
{
  int err = 0;
  struct msghdr msg;
  struct iovec iov;

  iov.iov_base = src;
  iov.iov_len = len;

  msg.msg_name = NULL;
  msg.msg_namelen = 0;
  msg.msg_iov = &iov;
  msg.msg_iovlen = 1;
  msg.msg_control = NULL;
  msg.msg_controllen = 0;
  msg.msg_flags = 0;
  err = recvmsg(sockfd, &msg, 0);
  if (err == -1) {
    perror("recvmsg operation err");
    goto out;
  }
  printf("recvmsg data: \n");
  print_hex_dump(src, len);

out:
  err = errno;
  return err;
}

int main(void)
{
	int opfd, tfmfd, err =0;
    char plaintext_buf[] = {0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77, 0x88, 
                        0x99, 0x00, 0xaa, 0xbb, 0xcc, 0xdd, 0xee, 0xff, 0x00, 0x11};
    char key_buf[] = {0xff, 0xd7, 0x40, 0x57, 0x47, 0x68, 0x5e, 0xd6, 0xe0, 
                        0x0b, 0xc6, 0x82, 0xa7, 0x72, 0x86, 0x09};
    char encrypt_buf[32];
    char decrypt_buf[32];

	struct sockaddr_alg sa = {
		.salg_family = AF_ALG,
		.salg_type = "skcipher",
		.salg_name = "AES128_ECB_CLR_CLR"
	};

	tfmfd = socket(AF_ALG, SOCK_SEQPACKET, 0);
    if(tfmfd == -1) {
        perror("socket err");
        goto socket_err;
    }

	err = bind(tfmfd, (struct sockaddr *)&sa, sizeof(sa));
    if(err) {
        perror("bind error");
        goto bind_err;
    }

    printf("plaintext buf: \n");
    print_hex_dump(plaintext_buf, sizeof(plaintext_buf));

    /* setkey, we need to set key before connect */
    err = setkey(tfmfd, key_buf, sizeof(key_buf));
    if(err)
        goto setkey_err;

	opfd = accept(tfmfd, NULL, 0);
    if(opfd == -1) {
        perror("accept err!");
        goto accept_err;
    }
    /* set iv */

    /* set encryption */
    err = sendmsg_to_cipher(opfd, ALG_SET_OP,
                      ALG_OP_ENCRYPT, plaintext_buf, sizeof(plaintext_buf));
    if(err)
        goto sendmsg_err;

    err = recvmsg_from_cipher(opfd, encrypt_buf, sizeof(plaintext_buf));
    if (err)
      goto recvmsg_err;

    /* set decryption */
    err = sendmsg_to_cipher(opfd, ALG_SET_OP, ALG_OP_DECRYPT, encrypt_buf,
                            sizeof(plaintext_buf));
    if (err)
      goto sendmsg_err;

    err = recvmsg_from_cipher(opfd, decrypt_buf, sizeof(plaintext_buf));
    if (err)
      goto recvmsg_err;

recvmsg_err:
sendmsg_err:
    close(opfd);
accept_err:
setkey_err:
bind_err:
    close(tfmfd);
socket_err:
    return err;
}
```

### 3. openssl
#### 3.1. afalg engine
我们再1.1.0 版本后的openssl/engines 可以找到e_afalg.c 模块。在1.1.1c 版本中支持的afalg 只有三种：
- aes_128_cbc
- aes_192_cbc
- aes_256_cbc

```c
static int afalg_ciphers(ENGINE *e, const EVP_CIPHER **cipher,
                         const int **nids, int nid)
{
    int r = 1;

    if (cipher == NULL) {
        *nids = afalg_cipher_nids;
        return (sizeof(afalg_cipher_nids) / sizeof(afalg_cipher_nids[0]));
    }

    switch (nid) {
    case NID_aes_128_cbc:
    case NID_aes_192_cbc:
    case NID_aes_256_cbc:
        *cipher = afalg_aes_cbc(nid);
        break;
    default:
        *cipher = NULL;
        r = 0;
    }
    return r;
}
```

openssl 内部的afalg engine 其实与自己写的userspace code 类似，都是使用netlink AF_ALG family socket。

```c
static ossl_inline void afalg_set_op_sk(struct cmsghdr *cmsg,
                                   const ALG_OP_TYPE op)
{
    cmsg->cmsg_level = SOL_ALG;
    cmsg->cmsg_type = ALG_SET_OP;
    cmsg->cmsg_len = CMSG_LEN(ALG_OP_LEN);
    memcpy(CMSG_DATA(cmsg), &op, ALG_OP_LEN);
}

static void afalg_set_iv_sk(struct cmsghdr *cmsg, const unsigned char *iv,
                            const unsigned int len)
{
    struct af_alg_iv *aiv;

    cmsg->cmsg_level = SOL_ALG;
    cmsg->cmsg_type = ALG_SET_IV;
    cmsg->cmsg_len = CMSG_LEN(ALG_IV_LEN(len));
    aiv = (struct af_alg_iv *)CMSG_DATA(cmsg);
    aiv->ivlen = len;
    memcpy(aiv->iv, iv, len);
}

static ossl_inline int afalg_set_key(afalg_ctx *actx, const unsigned char *key,
                                const int klen)
{
    int ret;
    ret = setsockopt(actx->bfd, SOL_ALG, ALG_SET_KEY, key, klen);

    return 1;
}

static int afalg_create_sk(afalg_ctx *actx, const char *ciphertype,
                                const char *ciphername)
{
    struct sockaddr_alg sa;
    int r = -1;

    actx->bfd = actx->sfd = -1;

    memset(&sa, 0, sizeof(sa));
    sa.salg_family = AF_ALG;
    strncpy((char *) sa.salg_type, ciphertype, ALG_MAX_SALG_TYPE);
    sa.salg_type[ALG_MAX_SALG_TYPE-1] = '\0';
    strncpy((char *) sa.salg_name, ciphername, ALG_MAX_SALG_NAME);
    sa.salg_name[ALG_MAX_SALG_NAME-1] = '\0';

    actx->bfd = socket(AF_ALG, SOCK_SEQPACKET, 0);

    r = bind(actx->bfd, (struct sockaddr *)&sa, sizeof(sa));

    actx->sfd = accept(actx->bfd, NULL, 0);

    return 1;
}
```

openssl 中体现的优点在于：
- 使用EVP 抽象统一all engines， 比如afalg, crypdev 等
- 使用async io
- 支持zero copy (vmsplice(), splice() 函数使用)

#### 3.2. performance
使用openssl speed 模块可以测试速度。
>time openssl speed -evp aes-128-cbc -elapsed 
>time openssl speed -evp aes-128-cbc -elapsed -engine afalg

Method | Result
:-: | :-
Software Crypto | type 16 bytes 64 bytes 256 bytes 1024 bytes 8192 bytes 16384 bytes <br> <font color=red>aes-128-cbc 13681.29k 16959.06k 18170.71k 18484.91k 18590.38k 18573.99k</font> <br><br> real 0m 18.67s <br> user 0m 18.10s <br> sys 0m 0.19s <br>
Hardware Crypto | type 16 bytes 64 bytes 256 bytes 1024 bytes 8192 bytes 16384 bytes <br><font color=red>	aes-128-cbc 293.11k 1052.46k 4073.64k 11785.59k 27314.86k 30938.45k</font> <br><br> real 0m 18.24s <br> user 0m 0.73s<br> sys 0m 10.80s


### 参看资料

[User Space Interface](https://01.org/linuxgraphics/gfx-docs/drm/crypto/userspace-if.html)

[Crypto API (Linux)
](https://blog.csdn.net/yazhouren/article/details/53035690)

[userspace if_alg example code](https://blog.csdn.net/yazhouren/article/details/53035742)

[libkcapi, crypto user space library](http://www.chronox.de/libkcapi.html)