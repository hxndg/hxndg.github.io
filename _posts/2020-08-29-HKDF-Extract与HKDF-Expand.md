---
layout:     post   				    # 使用的布局（不需要改）
title:      HKDF-Extract与HKDF-Expand
subtitle:   TLS1.3中的基础知识 #副标题
date:       2020-08-27 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - TLS1.3
---

# HKDF-Extract与HKDF-Expand

## 前言
写TLS1.3的协议解析自动机的时候，不单实现了cavium卡的流程，还得实现软实现。所以硬生生对着HKDF的RFC和OPENSSL的源代码吃了一遍。做的时候还有些疑问，不知道有没有本科生看，希望我写的能直接给本科生看。

## HKDF-Extract与HKDF-Expand的作用
做密钥衍生的时候常常需要根据初始密钥材料（initial keying  material）产生符合特定长度要求，密码学安全标准的新密钥的需求。因此RFC5889定义了一种基于HMAC的密钥衍生函数（KDF），合起来就是HKDF（KDF前面加一个H）。HKDF-Extract与HKDF-Expand就是一枚硬币的两面，一体双生，两者结合才能产生安全的新密钥。

### HKDF-Extract
HKDF-Extract从初始密钥材料中“拽（extract）”出固定长度的伪随机密钥K(K就是个代号)，
### HKDF-Expand
HKDF-Expand过程，负责将伪随机密钥K“拉（expand）”也就是拓展为多份附加伪随机密钥，也就是KDF的输出。

## HKDF-Extract与HKDF-Expand的RFC
RFC定义的非常简单了。

### HKDF-Extract
```
HKDF-Extract(salt, IKM) -> PRK

   Options:
      Hash     a hash function; HashLen denotes the length of the
               hash function output in octets

   Inputs:
      salt     optional salt value (a non-secret random value);
               if not provided, it is set to a string of HashLen zeros.
      IKM      input keying material

   Output:
      PRK      a pseudorandom key (of HashLen octets)

   The output PRK is calculated as follows:

   PRK = HMAC-Hash(salt, IKM)
```
可以看到，HKDF-Extract的过程非常简单，本质上就是初始密钥材料加盐做一次Hmac-Hash。如果没有提供盐的话，就是遗传长度为hashLen的0字符串。
### HKDF-Expand
```
HKDF-Expand(PRK, info, L) -> OKM

   Options:
      Hash     a hash function; HashLen denotes the length of the
               hash function output in octets
			   
   Inputs:
      PRK      a pseudorandom key of at least HashLen octets
               (usually, the output from the extract step)
      info     optional context and application specific information
               (can be a zero-length string)
      L        length of output keying material in octets
               (<= 255*HashLen)

   Output:
      OKM      output keying material (of L octets)

   The output OKM is calculated as follows:

   N = ceil(L/HashLen)
   T = T(1) | T(2) | T(3) | ... | T(N)
   OKM = first L octets of T

   where:
   T(0) = empty string (zero length)
   T(1) = HMAC-Hash(PRK, T(0) | info | 0x01)
   T(2) = HMAC-Hash(PRK, T(1) | info | 0x02)
   T(3) = HMAC-Hash(PRK, T(2) | info | 0x03)
   ...
```
“拉”的过程略显复杂，我们下面按假设来做操作：
+ 首先根据需要输出数据的长度来判断要迭代多少次，比方说需要输出长度为129长度的密钥结果，hashLen为32字节。那么就需要叠加出来129/32再向上取整的结果，也就是5块。
+ 假设五个块分别为 T(1),T(2),T(3),T(4),T(5)。首先需要虚构一个T(0)出来，T(0)为空字符串，也就是""。然后每次将T(n-1)拼接上info再拼接上序号，和RPK(Extract的结果)一次做Hmac-Hash迭代出T(n)。公式如下
```math
T(N) = HMAC-Hash(PRK, T(N-1) | info | N)
```
+ 迭代计算够了以后，把每个输出的结果拼接起来，也就是T(1)|T(2)|T(3)...|T(N)取目标长度就拿到了输出的密钥了。这里我们是拼接T(1)到T(5)，取前129字节即可。
## HKDF-Extract与HKDF-Expand的代码实现
代码实现的基础是实现HMAC-hash，这个代码不多讲，属于基础知识。以后单独摘出来说。下面的代码直接抄的openssl的，我司的代码和openssl非常相似（废话，一样的做法必然相似啊）
### HKDF-Extract
```c
static unsigned char *HKDF_Extract(const EVP_MD *evp_md,
                                   const unsigned char *salt, size_t salt_len,
                                   const unsigned char *key, size_t key_len,
                                   unsigned char *prk, size_t *prk_len)
{
        unsigned int tmp_len;

        if (!HMAC(evp_md, salt, salt_len, key, key_len, prk, &tmp_len)) {
				return NULL;
		}
        

        *prk_len = tmp_len;
        return prk;
}
```
简单说说，salt就是参与计算的盐,salt_len为盐长度，如果盐为NULL或者盐长度为0，就会被初始化为空字符串即`static const unsigned char dummy_key[1] = {'\0'};`,key就是输入的初始密钥材料，利用它计算出来伪随机密钥（PRK）。这里唯一注意的就是prk和prk_len是存储结果的。
### HKDF-Expand

```c
static unsigned char *HKDF_Expand(const EVP_MD *evp_md,
                                  const unsigned char *prk, size_t prk_len,
                                  const unsigned char *info, size_t info_len,
                                  unsigned char *okm, size_t okm_len)
{
    HMAC_CTX *hmac;
    unsigned char *ret = NULL;
    unsigned int i;
    unsigned char prev[EVP_MAX_MD_SIZE];
    size_t done_len = 0, dig_len = EVP_MD_size(evp_md);

    size_t n = okm_len / dig_len;  //计算需要产生几块T(x)，如果像输出的结果不是hashLen的整数倍，需要向上取整。
    if (okm_len % dig_len)
        n++;
		
    if (n > 255 || okm == NULL)		//如果输出的地址或者长度太长，直接就当失败了。
        return NULL;

    if ((hmac = HMAC_CTX_new()) == NULL) //初始化HMAC失败那就算了，“毁灭吧，赶紧的”
        return NULL;

    if (!HMAC_Init_ex(hmac, prk, prk_len, evp_md, NULL)) //初始化下hmac使用的函数
        goto err;

    for (i = 1; i <= n; i++) {
        size_t copy_len;
        const unsigned char ctr = i;

        if (i > 1) {
            if (!HMAC_Init_ex(hmac, NULL, 0, NULL, NULL))
                goto err;

            if (!HMAC_Update(hmac, prev, dig_len)) //如果不是第一次计算，也就是由T(0)计算T(1)，那么需要把T(N-1)作为数据拼接到计算中，失败就直接算了
                goto err;
        }

        if (!HMAC_Update(hmac, info, info_len))		//可以拼接上info信息了，失败就直接算了
            goto err;

        if (!HMAC_Update(hmac, &ctr, 1))		//可以拼接上计数器序号了，失败就直接算了
            goto err;

        if (!HMAC_Final(hmac, prev, NULL))		//好，算一次hmac，然后就从T(N-1)得到了T(N)了。
            goto err;
												//下面的结果就是不断把T(x)拼接起来的过程，边拷贝边叠加长度
        copy_len = (done_len + dig_len > okm_len) ?
                       okm_len - done_len :
                       dig_len;

        memcpy(okm + done_len, prev, copy_len);

        done_len += copy_len;
    }
    ret = okm;									//这就是输出的结果

 err:
    OPENSSL_cleanse(prev, sizeof(prev));
    HMAC_CTX_free(hmac);
    return ret;
}
```

具体流程我就不提了，如果你看懂了“HKDF-Extract与HKDF-Expand的RFC”那章，那么这个实现可以说是非常简单了。
## HKDF-Extract与HKDF-Expand的一些疑问
在看RFC的时候，主要由两个疑问
+ 为什么要将HKDF分为两个部分，Extract和Expand？因为初始密钥材料可能并不是信息分布合理的，攻击者可能掌握部分初始密钥材料的信息或者可以操纵里面的一部分信息。所以使用extract流程来将分散的信息熵凝聚成为一个短的，符合密码学安全的伪随机密钥。如果初始密钥材料已经足够随机，那可以不进行extract操作的。第二个过程expand没什么好说的了，负责将筛选过的伪随机密钥拓展为目标长度。
+ 为什么要使用HKDF将作为标准的密钥衍生流程？直接用hash等不行吗？单纯从结果来看，可以直接使用hash等算法。但是除了上面安全方面考虑的原因，还有一个是因为这样既安全又标准，可以作为一种灵活的标准模块参与到计算当中去。是一种模块话的设计。

## 结尾
写到这里差不多就可以结束了，TLS这块还有啥不明白的直接告诉我就成了
![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)
