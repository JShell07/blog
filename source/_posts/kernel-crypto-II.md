---
title: kernel_crypto_II
date: 2019-08-06 20:04:59
tags: cryptographic
categories: drivers
---

### 1. cipher 基础
在Kernel crytographic 的cipher 还有如下几类：
- ablkcipher_tfm(asynchronous multi-block cipher transform)  
- blkcipher_tfm(synchronous multi-block cipher transform)   
- cipher_tfm(Single block cipher transform)

更为安全的是使用ahead 加密形式， 可以见此篇文章。[什么是AEAD加密](https://zhuanlan.zhihu.com/p/28566058)

<!--more-->

在kernel中cipher 都为skcipher(Symmetric Key cipher, 对称性加密算法)

后面我们以kernel/drivers/crypto/atmel-aes.c 进行分析ablkcipher 的驱动。

### 2. cipher alg register
atmel aes 驱动初始流程可见下图：
![driver init flow](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/aes-init-flow.png)

```c
static struct crypto_alg aes_algs[] = {
{
	.cra_name		= "ecb(aes)",
	.cra_driver_name	= "atmel-ecb-aes",
	.cra_priority		= 100,
	.cra_flags		= CRYPTO_ALG_TYPE_ABLKCIPHER | CRYPTO_ALG_ASYNC,
	.cra_blocksize		= AES_BLOCK_SIZE,
	.cra_ctxsize		= sizeof(struct atmel_aes_ctx),
	.cra_alignmask		= 0xf,
	.cra_type		= &crypto_ablkcipher_type,
	.cra_module		= THIS_MODULE,
	.cra_init		= atmel_aes_cra_init,
	.cra_exit		= atmel_aes_cra_exit,
	.cra_u.ablkcipher = {
		.min_keysize	= AES_MIN_KEY_SIZE,
		.max_keysize	= AES_MAX_KEY_SIZE,
		.setkey		= atmel_aes_setkey,
		.encrypt	= atmel_aes_ecb_encrypt,
		.decrypt	= atmel_aes_ecb_decrypt,
	}
},
{
	.cra_name		= "cbc(aes)",
	.cra_driver_name	= "atmel-cbc-aes",
	.cra_priority		= 100,
	.cra_flags		= CRYPTO_ALG_TYPE_ABLKCIPHER | CRYPTO_ALG_ASYNC,
	.cra_blocksize		= AES_BLOCK_SIZE,
	.cra_ctxsize		= sizeof(struct atmel_aes_ctx),
	.cra_alignmask		= 0xf,
	.cra_type		= &crypto_ablkcipher_type,
	.cra_module		= THIS_MODULE,
	.cra_init		= atmel_aes_cra_init,
	.cra_exit		= atmel_aes_cra_exit,
	.cra_u.ablkcipher = {
		.min_keysize	= AES_MIN_KEY_SIZE,
		.max_keysize	= AES_MAX_KEY_SIZE,
		.ivsize		= AES_BLOCK_SIZE,
		.setkey		= atmel_aes_setkey,
		.encrypt	= atmel_aes_cbc_encrypt,
		.decrypt	= atmel_aes_cbc_decrypt,
	}
},
};
```

### 2. encrypt flow
每当encryp 时， 我们将使用ablkcipher_enqueue_request() 
把`ablkcipher_request *req` 添加到队列上。在判断HW engine 是空闲可用时，再从queue 上取出并进行HW 的加解密操作。decrypt 的流程如此类似，区别在于设定HW 的mode 设定。

![hw encrypt flow](https://raw.githubusercontent.com/JShell07/images/master/kernel_crypto/aes-hw-encrypt-flow.png)

### 2.1. data structure
```c
struct crypto_queue {
	struct list_head list;
	struct list_head *backlog;
	unsigned int qlen;
	unsigned int max_qlen;
};

struct ablkcipher_request {
	struct crypto_async_request base;
	unsigned int nbytes;
	void *info;
	struct scatterlist *src;
	struct scatterlist *dst;
	void *__ctx[] CRYPTO_MINALIGN_ATTR;
};
```

在中断完成后，atmel_aes_done_task（）-> atmel_aes_finish_req() -> req->base.complete(&req->base, err) 这样会调用到complete（） 完成函数，实现了异步的通知行为。

```c
static void atmel_aes_done_task(unsigned long data)
{
	struct atmel_aes_dev *dd = (struct atmel_aes_dev *) data;
	int err;

	if (!(dd->flags & AES_FLAGS_DMA)) {
		atmel_aes_read_n(dd, AES_ODATAR(0), (u32 *) dd->buf_out,
				dd->bufcnt >> 2);

		if (sg_copy_from_buffer(dd->out_sg, dd->nb_out_sg,
			dd->buf_out, dd->bufcnt))
			err = 0;
		else
			err = -EINVAL;

		goto cpu_end;
	}

	err = atmel_aes_crypt_dma_stop(dd);

	err = dd->err ? : err;

	if (dd->total && !err) {
		if (dd->flags & AES_FLAGS_FAST) {
			dd->in_sg = sg_next(dd->in_sg);
			dd->out_sg = sg_next(dd->out_sg);
			if (!dd->in_sg || !dd->out_sg)
				err = -EINVAL;
		}
		if (!err)
			err = atmel_aes_crypt_dma_start(dd);
		if (!err)
			return; /* DMA started. Not fininishing. */
	}

cpu_end:
	atmel_aes_finish_req(dd, err);
	atmel_aes_handle_queue(dd, NULL);
}

static void atmel_aes_finish_req(struct atmel_aes_dev *dd, int err)
{
	struct ablkcipher_request *req = dd->req;

	clk_disable_unprepare(dd->iclk);
	dd->flags &= ~AES_FLAGS_BUSY;

	req->base.complete(&req->base, err);
}
```

我们可以参看kernel/crypto/testmgr.c 中的测试代码，它的使用案例。

>init_completion(&result.completion);
ablkcipher_request_alloc(tfm, GFP_KERNEL);
ablkcipher_request_set_callback(req,CRYPTO_TFM_REQ_MAY_BACKLOG, tcrypt_complete, &result);
crypto_ablkcipher_clear_flags(tfm, ~0);
crypto_ablkcipher_setkey(tfm, template[i].key, template[i].klen);
ablkcipher_request_set_crypt(req, sg, sgout, template[i].ilen, iv);
crypto_ablkcipher_encrypt(req);
crypto_ablkcipher_decrypt(req);

```c
static void tcrypt_complete(struct crypto_async_request *req, int err)
{
	struct tcrypt_result *res = req->data;

	if (err == -EINPROGRESS)
		return;

	res->err = err;
	complete(&res->completion);
}

static int __test_skcipher(struct crypto_ablkcipher *tfm, int enc,
			   struct cipher_testvec *template, unsigned int tcount,
			   const bool diff_dst, const int align_offset)
{
	const char *algo =
		crypto_tfm_alg_driver_name(crypto_ablkcipher_tfm(tfm));
	unsigned int i, j, k, n, temp;
	char *q;
	struct ablkcipher_request *req;
	struct scatterlist sg[8];
	struct scatterlist sgout[8];
	const char *e, *d;
	struct tcrypt_result result;
	void *data;
	char iv[MAX_IVLEN];
	char *xbuf[XBUFSIZE];
	char *xoutbuf[XBUFSIZE];
	int ret = -ENOMEM;

	init_completion(&result.completion);

	req = ablkcipher_request_alloc(tfm, GFP_KERNEL);
	
	ablkcipher_request_set_callback(req, CRYPTO_TFM_REQ_MAY_BACKLOG,
					tcrypt_complete, &result);

	j = 0;
	for (i = 0; i < tcount; i++) {
		if (template[i].np && !template[i].also_non_np)
			continue;

		if (template[i].iv)
			memcpy(iv, template[i].iv, MAX_IVLEN);
		else
			memset(iv, 0, MAX_IVLEN);

		j++;
		

		data = xbuf[0];
		data += align_offset;
		memcpy(data, template[i].input, template[i].ilen);
		crypto_ablkcipher_clear_flags(tfm, ~0);
		ret = crypto_ablkcipher_setkey(tfm, template[i].key,
					       template[i].klen);

		sg_init_one(&sg[0], data, template[i].ilen);

		ablkcipher_request_set_crypt(req, sg, (diff_dst) ? sgout : sg,
					     template[i].ilen, iv);
		ret = enc ? crypto_ablkcipher_encrypt(req) :
			    crypto_ablkcipher_decrypt(req);

		switch (ret) {
		case 0:
			break;
		case -EINPROGRESS:
		case -EBUSY:
			ret = wait_for_completion_interruptible(
				&result.completion);
			if (!ret && !((ret = result.err))) {
				reinit_completion(&result.completion);
				break;
			}
			/* fall through */
		default:
			pr_err("alg: skcipher%s: %s failed on test %d for %s: ret=%d\n",
			       d, e, j, algo, -ret);
			goto out;
		}

		q = data;
		if (memcmp(q, template[i].result, template[i].rlen)) {
			pr_err("alg: skcipher%s: Test %d failed on %s for %s\n",
			       d, j, e, algo);
			hexdump(q, template[i].rlen);
			ret = -EINVAL;
			goto out;
		}
	}
	ret = 0;
out:
	ablkcipher_request_free(req);
	return ret;
}
```
### 参看资料
[3.18.11/drivers/crypto/atmel-aes.c](https://elixir.bootlin.com/linux/v3.18.11/source/drivers/crypto/atmel-aes.c)