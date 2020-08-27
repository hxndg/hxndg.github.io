---
layout:     post   				    # 使用的布局（不需要改）
title:      TLS1.3 密钥衍生计算方法和功能
subtitle:   Hello World, Hello Blog #副标题
date:       2020-08-27 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - TLS1.3
---

#TLS1.3 密钥衍生计算方法和功能
##前言
Tls1.3的RFC（RFC8446）中规定了各种密钥的算法和用途，虽然我打算在不久的将来翻译TLS1.3的RFC，出于方便的考虑，还是简单讲讲TLS1.3中的各种密钥计算方法。因为RFC的密钥衍生流程中没有画出PSK的计算过程，我也会在本文附加PSK的计算流程如果文章帮到了你/或者依然存在一些问题，欢迎给我一些反馈。

##密钥衍生流程
###密钥衍生流程中一些计算方法
密钥衍生的流程依赖于RFC5869定义的HKDF-Extract和HKDF-Expand操作流程，这两个流程结合到一起才算是从密钥原始材料（key material)衍生出安全的符合长度需求的密钥。TLS1.3定义了一个HKDF-Expand-Label即带衍生标签的HKDF-Expand操作，这个操作的流程是：
>Derive-Secret(Secret, Label, Messages) =
HKDF-Expand-Label(Secret, Label,Transcript-Hash(Messages), Hash.length) =
HKDF-Expand(Secret, HkdfLabel, Hash.length)

如果你知道HKDF-Expand的操作（我会不久翻译RFC5869并且讲解HKDF的两个操作），问题就变成了HkdfLabel是如何拼接的。按照RFC，HkdfLabel的拼接方式如下：
>struct {
           uint16 length = Length;
           opaque label<7..255> = "tls13 " + Label;
           opaque context<0..255> = Context;
       } HkdfLabel;

首先拼接一个uint16_t的“长度”，也就是两个字节，该“长度”的数值等于当前算法使用的hash算法的长度，比方说你使用ciphersuite是TLS_AES_256_GCM_SHA384，所以该“长度的数值”等于SHA384计算结果的长度，也就是384/8=48字节。之后先拼接上一个uint8_t的“字符串长度”，这个长度用来标识整个字符串的长度，包括"tls1.3"+Label的长度，然后附加上字符串"tls13"（我这里附加的时候可不包括两个双引号啊），再拼接上Label，比方说你此时在计算复用主密钥(resumption_master_secret)，那么这时候的Label就是"res master"。最后拼接一个uint8_t的Context长度，再拼接Context。Context一般是对消息做hash计算得出的。
###密钥衍生流程及每个密钥的作用
这部分我是直接粘贴的TLS1.3 RFC的内容，见下，由于翻译可能产生争议，这部分我用的是原图。
![密钥衍生流程](https://wx1.sbimg.cn/2020/08/27/69eOR.png)
密钥衍生流程中有两个用于计算的基础密钥，分别是PSK和ECDHE secret。PSK通过上次连接由复用主密钥计算得出/带外数据传输建立，(EC)DHE密钥则是完整握手时协商产生。这两个密钥并不一定每次都存在，比方说完整握手的情况下，计算初期密钥（Early Secret）的时候可能不存在PSK，那么PSK就由一串长度等同于当前ciphersuite使用的hash算法结果的长度的0x00代替，比方说ciphersuite是TLS_AES_256_GCM_SHA384，那么PSK就用长度等于48的0x00内容替代。我在写TLS1.3 自动机的时候就直接写了个
>static const unsigned char default_zeros[EVP_MAX_MD_SIZE];

来替代。

流程当中HKDF-Extract顶部的数据充当“盐”的角色，左侧是原始密钥材料（IKM），其输出结果为衍生出来的密钥。我们熟悉了HKDF-Extract和Derive-Secret之后，就会发现TLS1.3中是结合了HKDF-Extract和HKDF-Expand一起计算出来各种密钥的，这点比TLS1.2更符合RFC的要求，更安全。

还有一点值得注意，整个密钥衍生流程是不可以跳过任何一步的，每一步都必须进行（这里的必须进行只针对需要的数据，比方说你不开复用，那么就没必要计算PSK，复用主密钥之类的）。举个简单的例子，完整握手的情况下，仍然需要计算初期密钥，计算的方式就是HKDF-Extract(0, 0)。

最后一个需要注意的是，我们计算出来的密钥并不直接参与到会话/握手当中去，是要经过一论演算的。也就是说从secret到key是有一个过程的，举个例子说，客户端握手密钥（client_handshake_traffic_secret）计算到客户端会话key是有个过程的，如下：
> client_handshake_write_key = HKDF-Expand-Label(client_handshake_traffic_secret, "key", "", key_length)
  client_handshake_write_iv  = HKDF-Expand-Label(client_handshake_traffic_secret, "iv", "", iv_length)

我这里之所以写client_handshake_write_key的时候加了write是因为客户端发送握手消息使用的key和客户端接受握手消息（服务端发送握手消息）的key是不一样的。这一点在TLS1.2中也是如此。

###PSK的计算过程
这次只简单写写PSK的计算过程，复用时如何计算binder value，恢复会话的操作以后单独写。
PSK由复用主密钥（resumption_master_secret）计算得出，公式如下：
>PSK = 
HKDF-Expand-Label(resumption_master_secret,"resumption", ticket_nonce, Hash.length)

这里复用主密钥不需要多说，而ticket_nonce需要简单说说：ticket_nonce每个连接只有一个，其值应当每个都不一样。ticket_nonce需要明文发给客户端，让客户端计算出来PSK。包含ticket_nonce的报文就是new session ticket报文，简写为NST报文。

直接对照“密钥衍生流程中一些计算方法”就可以清晰的计算出来PSK，实际上从PSK到HMAC key，再到binder key的计算过程非常简单，完全可以一次算完。为了节省复用时的时间，可以直接计算存储binder key。但是由于涉及到操作都是hmac/hash，所以本质上时间减少的很少。这东西我第一个想出来以后被我司拿去申请了专利，还给了点奖金。
###密钥衍生中的一些细节
这里写的东西是我当初踩到的一些坑，有的我注意到避开了，有的没注意到踩了进去。
1 计算HKDF-Extract和HKDF-Extract时，需要注意，对于空消息的hash结果并不为空。其结果时有值的。因为忘了hash计算的流程，就容易踩这个坑。目前tls1.3的cipher只有两种hash长度一个是sha384的48,另一个时sha256的32。直接把当时写的结果贴上来了，各位也不用自己算了。
>uint8_t zero_sha384_hash[TLSV13_SHA384_HASH_LEN] = {0x38, 0xb0, 0x60, 0xa7,0x51, 0xac, 0x96, 0x38, 0x4c, 0xd9, 0x32, 0x7e,0xb1, 0xb1, 0xe3, 0x6a, 0x21, 0xfd, 0xb7, 0x11,0x14, 0xbe, 0x07, 0x43, 0x4c, 0x0c, 0xc7, 0xbf,0x63, 0xf6, 0xe1, 0xda, 0x27, 0x4e, 0xde, 0xbf,0xe7, 0x6f, 0x65, 0xfb, 0xd5, 0x1a, 0xd2, 0xf1,0x48, 0x98, 0xb9, 0x5b}; 
uint8_t zero_sha256_hash[TLSV13_SHA256_HASH_LEN] = {0xe3, 0xb0, 0xc4, 0x42,0x98, 0xfc, 0x1c, 0x14, 0x9a, 0xfb, 0xf4, 0xc8,0x99, 0x6f, 0xb9, 0x24, 0x27, 0xae, 0x41, 0xe4,0x64, 0x9b, 0x93, 0x4c, 0xa4, 0x95, 0x99, 0x1b,0x78, 0x52, 0xb8, 0x55};

2 对于每个TLS1.3的连接，需要自己保存ticket_nonce。踩这个坑是因为写的自动机时异步，计算PSK时候用的ticket_nonce和写NST报文的时候的ticket_nonce可能会变化。因此需要每个连接自己保存自己的ticket_nonce。

###结尾
写到这里差不多就可以结束了，TLS这块还有啥不明白的直接告诉我就成了
![69zuK.jpg](https://wx1.sbimg.cn/2020/08/27/69zuK.jpg)
