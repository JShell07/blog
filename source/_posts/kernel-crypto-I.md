---
title: kernel_crypto_I
date: 2019-08-06 19:37:35
tags: cryptographic
categories: drivers
---

### 1. 基础
SE(Securty Engine) 提供encryption/decryption。
SE 常见使用如下形式加解密：
- cipher engine(密码引擎)  
- short messages（short message 指front message 的长度小于block cipher length）  
- residue（残余） technology (residue 指当我们最后的数据是小于cipher input block length 剩下的data)

术语 | 解释
:- | :-
明文| 原始信息
加密算法| 以密钥为参数，对明文进行多种置换和转换的规则和步骤，变换结果为密文。
密钥| 加密与解密算法的参数，直接影响对明文进行变换的结果
密文| 对明文进行变换的结果
解密算法| 加密算法的逆变换，以密文为输入、密钥为参数，变换结果为明文

<!--more-->

#### 1.1. 常见cipher engine 
- AES 128/192/256  
- DES  
- TDES(EEE/DDD/EDE/DED)

当然cipher 还有如下full chain(加密模式) 可以搭配, 例如AES_ECB, DES_CBC等。
- ECB 
- CBC  
- CTR  
- OFB  
- CFB   

例如ECB 模式： 
![ecb](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/ecb.png)

residue ecb_clr：
![residue](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/residue_ecb.png)

shortmessage ecb_clr:
![shortmessage](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/short_message_for%20ecb_with_clr.png)

cipher chain | CLR (short message) | XOR(IV1)(short message) | XOR(IV2)(short message) | CLR(residue) | RBT(residue) | CTS(residue) 
:- | :- | :- | :- | :- | :- | :- 
<font color=blue>AES_ECB</font> | <font color=red>O</font> | X | X | <font color=red>O</font> | X | X
<font color=blue>AES_CBC</font> | <font color=red>O</font> | X | X | <font color=red>O</font> | <font color=red>O</font> | <font color=red>O</font>
<font color=blue>AES_CTR</font> | X | <font color=red>O</font> | X | X | <font color=red>O</font> | <font color=red>O</font>
<font color=blue>AES_OFB</font> | <font color=red>O</font> | X | X | <font color=red>O</font> | X | X
<font color=blue>AES_CFB</font> | <font color=red>O</font> | X | X | <font color=red>O</font> | X | X
<font color=green>DES_ECB | <font color=red>O</font> | X | X | <font color=red>O</font> | X | X
<font color=green>DES_CBC | <font color=red>O</font> | X | X | <font color=red>O</font> | X | X
<font color=green>DES_CBC | X | <font color=red>O</font> | <font color=red>O</font> | X | <font color=red>O</font> | X
<font color=green>DES_CTR | <font color=red>O</font> | X | X | <font color=red>O</font> | X | X
<font color=green>DES_OFB | <font color=red>O | X | X | <font color=red>O | X | X
<font color=green>DES_CFB | <font color=red>O | X | X | <font color=red>O | X | X
<font color=orange>TES_ECB | <font color=red>O | X | X | <font color=red>O | X | X
<font color=orange>TES_CBC | <font color=red>O | X | X | <font color=red>O | X | X
<font color=orange>TES_CBC | X | <font color=red>O | <font color=red>O | X | <font color=red>O | X
<font color=orange>TES_CTR | <font color=red>O | X | X | <font color=red>O | X | X
<font color=orange>TES_OFB | <font color=red>O | X | X | <font color=red>O | X | X
<font color=orange>TES_CFB | <font color=red>O | X | X | <font color=red>O | X | X
__注__：
<font color=red>O</font>: legal
 X: illegal

#### 1.2. key(密钥)
硬件模块支持set key。有些还提供了CPU cannot access 的internal key 进一步保证安全性。因此，我们在加密时需要选择何种key。

![key selection](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/key_selection.png)

当然，internal key 是可以重新生成的，重新产生流程如下：
![internal key generate](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/internal_key_generate.png)

![internal key generate flow](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/internal_key_generate_flow.png)

#### 1.3. R/W 方式
cryptographic 一般支持三种方式的encryption or decrytion:
- pio  
- dma
- List + dma  

__List Mode__
每一entry 内容常见为：
- attribute (algorithm)  
- dma size  
- dma r/w addr  
  
![list entry](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/link_list.png)

__PIO Mode__
![pio encrypto flow](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/pio_encrypttion_flow.png)


### 2. Kernel Crypto
crypto 在kernel 中可以分为如下几类（）：
1. cipher (AES, DES, TDES等)
1.1. Symmetric cipher(ablkcipher, blkcipher, cipher)  
1.2. aead cipher(Authenticated Encryption with Associated Data )  
2. compress(zlib, lzo 等)  
3. digest  (摘要算法例如crc32, sha1, md5等)
4. random （软件层随机数）
5. hash (CAC, HMAC, XCBC, VMAC 等)

#### 2.1. kernel menuconfig
![menuconfig_core](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/menconfig_core.png)

![menuconfig_associate_data](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/menconfig_associated_data.png)

![menuconfig_cipher](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/menuconfig_cipher.png)

kernel 的crypto 子系统采用了分层的思想。作者使用crypto_alg 代表算法例如（AES_CBC, DES_CBC等）， crypto_tfm 代表用户实例化的对象，它包含了算法与处理逻辑。

>crt_u里面的回调函数和cra_u中的回调函数名称几乎一模一样，但是它们的层次不同，crt中的函数实现了一大类算法的运行逻辑，比如cipher中的des中的块应该怎么分割等等，虽然对于摘要算法，sha1或者别的什么的算法逻辑没有什么区别，但是对于cipher来讲就不是这样了，同一种算法可能拥有ecb，cbc，fcb等不同的模式，于是就来了个中间层，这个中间层就是上面的联合体crt_u。

#### 2.2. data structure
```c
struct crypto_alg {
	struct list_head cra_list;
	struct list_head cra_users;

	u32 cra_flags;
	unsigned int cra_blocksize;
	unsigned int cra_ctxsize;
	unsigned int cra_alignmask;

	int cra_priority;
	atomic_t cra_refcnt;

	char cra_name[CRYPTO_MAX_ALG_NAME];
	char cra_driver_name[CRYPTO_MAX_ALG_NAME];

	const struct crypto_type *cra_type;

	union {
		struct ablkcipher_alg ablkcipher;
		struct aead_alg aead;
		struct blkcipher_alg blkcipher;
		struct cipher_alg cipher;
		struct compress_alg compress;
		struct rng_alg rng;
	} cra_u;

	int (*cra_init)(struct crypto_tfm *tfm);
	void (*cra_exit)(struct crypto_tfm *tfm);
	void (*cra_destroy)(struct crypto_alg *alg);
	
	struct module *cra_module;
};

/*
 * Transforms: user-instantiated objects which encapsulate algorithms
 * and core processing logic.  Managed via crypto_alloc_*() and
 * crypto_free_*(), as well as the various helpers below.
 */
struct crypto_tfm {
	u32 crt_flags;
	union {
		struct ablkcipher_tfm ablkcipher;
		struct aead_tfm aead;
		struct blkcipher_tfm blkcipher;
		struct cipher_tfm cipher;
		struct hash_tfm hash;
		struct compress_tfm compress;
		struct rng_tfm rng;
	} crt_u;

	void (*exit)(struct crypto_tfm *tfm);
	struct crypto_alg *__crt_alg;
	void *__crt_ctx[] CRYPTO_MINALIGN_ATTR;
};
```
#### 2.3. usage
kernel中crypto 相关源码在两个位置：
- ${kernel_src}/crypto （向Kernel或userspace提供的api） 
- ${kernel_src}/drivers/crypto （支持硬件加密的驱动）

tfm 的管理通过如下函数：
```c
crypto_alloc_tfm()
crypto_free_tfm()
```

在SE 驱动中，我们使用crypto_register_alg() 将他们添加到list中
```c
int crypto_register_algs(struct crypto_alg *algs, int count;
list_add(&alg->cra_list, &crypto_alg_list);

int crypto_unregister_algs(struct crypto_alg *algs, int count);
```
他们的使用关系大致如下：
>crypto API <—> crypto core <—> crypto_register_alg

### 参看资料
[常用加解密算法总结1-DES、TDES、3DES](https://blog.csdn.net/youyu_torch/article/details/78117530)

[Linux加密框架设计与实现(转)](https://www.cnblogs.com/hoys/archive/2013/03/25/2981612.html)

[linux内核cryto接口的实现以及与openssl的比较](https://blog.csdn.net/dog250/article/details/5561075)

[Linux Kernel Crypto API](https://01.org/linuxgraphics/gfx-docs/drm/crypto/)

[什么是AEAD加密](https://zhuanlan.zhihu.com/p/28566058)