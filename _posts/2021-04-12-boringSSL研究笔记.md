---
layout:     post   				    # 使用的布局（不需要改）
title:      boringSSL研究笔记
subtitle:    #副标题
date:       2021-04-12- 				# 时间
author:     大狗 						# 作者
header-img: img/post-bg-map.jpg
catalog: true 						# 是否归档
tags:								
    - 学习
---

# 2021-04-12-boringSSL研究笔记

boringSSL是google处于方便目的自己实现的SSL库，和openssl的通配性相比，boringSSL主要是定制型比较好，这几天我就打算研究下boringSSL和openSSL的差别。先从基本的数据结构出发，一部分的数据结构不言自明，就不多赘述了。有一点需要注意，想学习源码必须熟悉RFC！很多基础知识不会过多赘述，就是RFC的内容。

## 1 连接相关的数据结构

### 1.1 cipher数据结构

boringSSL取了cipher的通用name和具体的cipher id（和clienthello中保持一致），但是和openssl不一致的地方是boringSSL拆出来了通用属性，作为结构体里面的成员。可以看到，拆出来了秘钥交换算法，认证算法，对称加密算法，MAC算法，秘钥衍生算法。有一个很有趣的东西是`SSLCipherPreferenceList`，这个东西提供了prefertcert功能。头文件定义于internal.h文件，具体代码见下：

```c++
struct ssl_cipher_st {
  // name is the OpenSSL name for the cipher.
  const char *name;
  // standard_name is the IETF name for the cipher.
  const char *standard_name;
  // id is the cipher suite value bitwise OR-d with 0x03000000.
  uint32_t id;

  // algorithm_* determine the cipher suite. See constants below for the values.
  uint32_t algorithm_mkey;
  uint32_t algorithm_auth;
  uint32_t algorithm_enc;
  uint32_t algorithm_mac;
  uint32_t algorithm_prf;
};

BSSL_NAMESPACE_BEGIN

// Bits for |algorithm_mkey| (key exchange algorithm).
#define SSL_kRSA 0x00000001u
#define SSL_kECDHE 0x00000002u
// SSL_kPSK is only set for plain PSK, not ECDHE_PSK.
#define SSL_kPSK 0x00000004u
#define SSL_kGENERIC 0x00000008u

// Bits for |algorithm_auth| (server authentication).
#define SSL_aRSA 0x00000001u
#define SSL_aECDSA 0x00000002u
// SSL_aPSK is set for both PSK and ECDHE_PSK.
#define SSL_aPSK 0x00000004u
#define SSL_aGENERIC 0x00000008u

#define SSL_aCERT (SSL_aRSA | SSL_aECDSA)

// Bits for |algorithm_enc| (symmetric encryption).
#define SSL_3DES 0x00000001u
#define SSL_AES128 0x00000002u
#define SSL_AES256 0x00000004u
#define SSL_AES128GCM 0x00000008u
#define SSL_AES256GCM 0x00000010u
#define SSL_eNULL 0x00000020u
#define SSL_CHACHA20POLY1305 0x00000040u

#define SSL_AES (SSL_AES128 | SSL_AES256 | SSL_AES128GCM | SSL_AES256GCM)

// Bits for |algorithm_mac| (symmetric authentication).
#define SSL_SHA1 0x00000001u
// SSL_AEAD is set for all AEADs.
#define SSL_AEAD 0x00000002u

// Bits for |algorithm_prf| (handshake digest).
#define SSL_HANDSHAKE_MAC_DEFAULT 0x1
#define SSL_HANDSHAKE_MAC_SHA256 0x2
#define SSL_HANDSHAKE_MAC_SHA384 0x4

// SSL_MAX_MD_SIZE is size of the largest hash function used in TLS, SHA-384.
#define SSL_MAX_MD_SIZE 48
```

### 1.2 握手的基本数据结构

### 1.3 握手过程当中涉及到的数据结构

#### 1.3.1 transcript hash数据结构

transcript实际上就是报文的完整结构，除了TLS1.3的HELLO RETRY REQUEST会加上一些特殊的地方，其他都是报文。可以看到，GOOGLE把所有的跟HASH相关的动作都包裹到`SSLTranscript`结构中。但是从功能的角度来看，我觉得这个就属于C++的强行面向对象了，我个人很厌烦这一点，不像C那么灵活和彻底。

```c++
// SSLTranscript maintains the handshake transcript as a combination of a
// buffer and running hash.
class SSLTranscript {
 public:
  SSLTranscript();
  ~SSLTranscript();

  // Init initializes the handshake transcript. If called on an existing
  // transcript, it resets the transcript and hash. It returns true on success
  // and false on failure.
  bool Init();

  // InitHash initializes the handshake hash based on the PRF and contents of
  // the handshake transcript. Subsequent calls to |Update| will update the
  // rolling hash. It returns one on success and zero on failure. It is an error
  // to call this function after the handshake buffer is released.
  bool InitHash(uint16_t version, const SSL_CIPHER *cipher);

  // UpdateForHelloRetryRequest resets the rolling hash with the
  // HelloRetryRequest construction. It returns true on success and false on
  // failure. It is an error to call this function before the handshake buffer
  // is released.
  bool UpdateForHelloRetryRequest();

  // CopyToHashContext initializes |ctx| with |digest| and the data thus far in
  // the transcript. It returns true on success and false on failure. If the
  // handshake buffer is still present, |digest| may be any supported digest.
  // Otherwise, |digest| must match the transcript hash.
  bool CopyToHashContext(EVP_MD_CTX *ctx, const EVP_MD *digest);

  Span<const uint8_t> buffer() {
    return MakeConstSpan(reinterpret_cast<const uint8_t *>(buffer_->data),
                         buffer_->length);
  }

  // FreeBuffer releases the handshake buffer. Subsequent calls to
  // |Update| will not update the handshake buffer.
  void FreeBuffer();

  // DigestLen returns the length of the PRF hash.
  size_t DigestLen() const;

  // Digest returns the PRF hash. For TLS 1.1 and below, this is
  // |EVP_md5_sha1|.
  const EVP_MD *Digest() const;

  // Update adds |in| to the handshake buffer and handshake hash, whichever is
  // enabled. It returns true on success and false on failure.
  bool Update(Span<const uint8_t> in);

  // GetHash writes the handshake hash to |out| which must have room for at
  // least |DigestLen| bytes. On success, it returns true and sets |*out_len| to
  // the number of bytes written. Otherwise, it returns false.
  bool GetHash(uint8_t *out, size_t *out_len);

  // GetFinishedMAC computes the MAC for the Finished message into the bytes
  // pointed by |out| and writes the number of bytes to |*out_len|. |out| must
  // have room for |EVP_MAX_MD_SIZE| bytes. It returns true on success and false
  // on failure.
  bool GetFinishedMAC(uint8_t *out, size_t *out_len, const SSL_SESSION *session,
                      bool from_server);

 private:
  // buffer_, if non-null, contains the handshake transcript.
  UniquePtr<BUF_MEM> buffer_;
  // hash, if initialized with an |EVP_MD|, maintains the handshake hash.
  ScopedEVP_MD_CTX hash_;
};

```




## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)
