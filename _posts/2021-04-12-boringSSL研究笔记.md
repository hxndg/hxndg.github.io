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

BORING SSL和OPENSSL相比较使用的参数，命令行的选项之类的东西都非常类似。



## 0 技术调研和总览

### 0.1 如何做技术调研

下面是记录的阅读的一部分技术调研的笔记。

+ 技术调研的目的是什么，需求是什么？泛需求？核心需求是什么
+ 合理安排时间
+ 调整心态

最终呈现的结果，可以分为以下几个方面：

+ 核心需求是什么？想做什么事情？（这一部分可以和具体TLS相关写代码的人聊聊）
+ 从哪些渠道收集了哪些信息？
+ 做了哪些常识？
+ 如果找到了解决方案，那么此方案的前提条件，适用场景，会不会有潜在的问题。优缺点分析，
+ 如何落地？如果使用具体的功能，需要注意哪些方面

具体到本次的调研，要考察

+ 具体技术的新不新，稳不稳定，容不容易维护
+ 结果量化，从各个角度进行技术的量化
+ 多搜集信息，类似的项目做了什么？它们的利弊有哪些？比较这些技术的相似和不同，他们的优缺点有哪些？
+ 具体的技术如何落地？

具体到本次的调研，要关注：

+ 性能方面的优化手段，包括但不限于计算加速，复用优化
+ 安全性方面的改进，
+ 兼容性剪裁，我们需要做哪些操作才能提供兼容，并且提供灵活的使用方法

一些具体的手段：

+ throughtworks技术雷达

### 0.2 功能比较

由于单纯阅读代码并不能直接检测具体代码的差别，因此为了对比具体的差别打算从以下三个方面下手：

+ 一个是GITHUB上面BORGINSSL从2020年开始的PATCH，它们是哪个方面的PATCH，是功能性还是补丁性质？
+ 具体文档中提到的与openssl不同的地方，明显的优势是哪些？
+ 具体代码功能的阅读，代码不能说谎，因此最后是必然追踪到代码的。有多少个地方使用了这种方法？覆盖的程度有多少？
+ REVIEW BOARD的检查，应该可以对应到BUG



#### 0.2.1 功能比较（文档）

兼容性的角度需要从多个方面区别，比方说功能，代码，下面是功能的比较，我会从S_SERVER的属性进行比较。

| 功能名称                       | OPENSSL | BORING SSL |
| ------------------------------ | ------- | ---------- |
| THREAD-SAFE，都以pthread为基础 | ✔       | ✔          |
| TICKET-AEAD 加密               |         |            |
| SSL-RENEGOTIATE                | 支持    | 默认不支持 |
|                                |         |            |





|      功能名称      | TLS1 | TLS1.1 | TLS1.2 | TLS1.3 | TLS1 | TLS1.1 | TLS1.2 | TLS1.3 |
| :----------------: | :--: | :----: | :----: | :----: | :--: | :----: | :----: | :----: |
|   SESSION TICKET   |  ✔   |   ✔    |   ✔    |   ✔    |  ✖   |   ✖    |   ✖    |   ✔    |
|  SESSION ID CACHE  |  ✔   |   ✔    |   ✔    |   ✔    |  ✔   |   ✔    |   ✔    |   ✔    |
| PREFER CIPHER LIST |  ✖   |   ✖    |   ✖    |   ✖    |  ✔   |   ✔    |   ✔    |   ✔    |
| SINGLE-USE TICKET  |  ✖   |   ✖    |   ✖    |   ✖    |  ✖   |   ✖    |   ✖    |   ✖    |
| TLS 重协商默认允许 |  ✔   |   ✔    |   ✔    |   ✔    |  ✖   |   ✖    |   ✖    |   ✖    |
|                    |      |        |        |        |      |        |        |        |
|                    |      |        |        |        |      |        |        |        |
|                    |      |        |        |        |      |        |        |        |
|                    |      |        |        |        |      |        |        |        |





代码的兼容性

+ 很多OPENSSL的CTRL函数被提出按了，Some OpenSSL APIs are implemented with `ioctl`-style functions such as `SSL_ctrl` and `EVP_PKEY_CTX_ctrl`, combined with convenience macros,In BoringSSL, these macros have been replaced with proper functions. The underlying `_ctrl` functions have been removed.
+ 一部分的加解密代码变了，比方说EVP_PKEY_HMAC，变成了HMAC_*



具体的代码差别可以看https://boringssl.googlesource.com/boringssl/+/HEAD/PORTING.md链接



#### 0.2.2 功能比较（PATCH）

从patch来看，boringssl引入了以下方面

+ ECH(ESNI)
+ HPKE
+ EVP
+ AEAD，应用范围越来越广了，除了加密/认证，还有nonce产生
+ 对侧信道攻击的防御，比方说lucky13
+ 安全方面，移除了所有的不安全算法。比方说CBC SHA2
+ 开启c11支持，移除gcc4.9.0代码的检测
+ 具体实现代码的考虑：线程安全的考虑。因为内部hold锁，外部hold锁的原因
+ 具体实现代码的考虑：性能优化。减少内存使用 & 证书压缩
+ 具体实现代码的考虑：内存回收是否清零
+ 关注一下crypto/trust_token/trust_token.c的代码https://github.com/google/boringssl/commit/07827156c9ef185d3777e0f72e1afc44fa9cc2e0
+ 还要关注这个trust token，Update TrustTokenV2 to use VOPRFs and assemble RR.参考这个链接https://eprint.iacr.org/2020/072/20200324:214215



### 0.3 总体比较

从实现来看BORINGSSL作为GOOGLE为TLS提供的基础库，主要包含以下几个方面的精细化内容：

+ 安全方面：内容很多，基础的HPKE，HPKE的利用ECH，AEAD逐渐取代了过去的EThenMac渗透到的各种方面（包括SSL ticket，关键数据的存储等），TrustToken + PMBToken。使用TLS1.3覆盖原本的基础协议。SIPHASH
+ 标准化方面：AEAD与HKDF-EXTRACT & HKDF-EXPAND
+ 性能方面：性能方面的优化并不多，而且和上面的内容比较重合，比方说使用AEAD追求安全和性能的平衡。使用证书压缩来简短通信流量。使用TLS1.3来减少RTT。在TLS1.3中GOOGLE引入SESSION TICKET/SESSION ID机制来降低全局数据查询的时间(换言之QUIC)。使用长连接来降低秘钥交换的性能消耗（这条已经很多开始使用了）
+ 兼容性方面：

我们下面拆开说：

安全方面：

+ HPKE，混合公钥加密机制提供了
+ AEAD
+ TurstToken：取代第三方cookie
+ TLS1.3的大规模使用

标准化方面：

性能方面：

+ AEAD安全性和性能的平衡
+ 压缩证书，对于证书链比较长的证书，进行压缩降低数据传输时候消耗的数据包数量
+ 使用TLS1.3在完整情况下减少一个RTT时间
+ 使用TLS1.3的SESSION ID和SESSION TICKET机制减少握手秘钥的迭代和签名等信息的计算（严格来说是减少RTT的时间消耗+非对称操作的时间来提高性能）。需要注意的是对于TLS1.3之前的协议，复用机制能大大减少性能消耗（时间延迟）。
+ 前向安全，这个是性能优化当中一个不是很被注意的东西。前向安全的好处是获得服务器秘钥被第三方获取后，也不能解密具体的通信流量。当时前向安全要求服务器每次必须重新产生随机秘钥，因此带来很大的性能消耗。
+ 使用长连接来减少秘钥的性能消耗。

兼容性方面：











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

#### 1.2.1 TLS连接结构

可以看出来SSL_HANDSHAKE结构和具体的握手状态是相关的，具体的握手状态在自动机里面才会用到，先不过于关注这几个东西。BoringSSL添加了一个SSL_CONFIG结构用于完成握手之后给后面连接保存必要的信息。 `SSL`并不是线程安全的，任何时刻只能有一个线程使用该数据结构。具体的配置可以直接对`SSL_CTX`结构或者世界对`SSL`结构配置。实际上这个东西就是我们关注的对象

```c++
enum ssl_hs_wait_t {
  ssl_hs_error,
  ssl_hs_ok,
  ssl_hs_read_server_hello,
  ssl_hs_read_message,
  ssl_hs_flush,
  ssl_hs_certificate_selection_pending,
  ssl_hs_handoff,
  ssl_hs_handback,
  ssl_hs_x509_lookup,
  ssl_hs_channel_id_lookup,
  ssl_hs_private_key_operation,
  ssl_hs_pending_session,
  ssl_hs_pending_ticket,
  ssl_hs_early_return,
  ssl_hs_early_data_rejected,
  ssl_hs_read_end_of_early_data,
  ssl_hs_read_change_cipher_spec,
  ssl_hs_certificate_verify,
};

enum ssl_grease_index_t {
  ssl_grease_cipher = 0,
  ssl_grease_group,
  ssl_grease_extension1,
  ssl_grease_extension2,
  ssl_grease_version,
  ssl_grease_ticket_extension,
  ssl_grease_last_index = ssl_grease_ticket_extension,
};

enum tls12_server_hs_state_t {
  state12_start_accept = 0,
  state12_read_client_hello,
  state12_read_client_hello_after_ech,
  state12_select_certificate,
  state12_tls13,
  state12_select_parameters,
  state12_send_server_hello,
  state12_send_server_certificate,
  state12_send_server_key_exchange,
  state12_send_server_hello_done,
  state12_read_client_certificate,
  state12_verify_client_certificate,
  state12_read_client_key_exchange,
  state12_read_client_certificate_verify,
  state12_read_change_cipher_spec,
  state12_process_change_cipher_spec,
  state12_read_next_proto,
  state12_read_channel_id,
  state12_read_client_finished,
  state12_send_server_finished,
  state12_finish_server_handshake,
  state12_done,
};

enum tls13_server_hs_state_t {
  state13_select_parameters = 0,
  state13_select_session,
  state13_send_hello_retry_request,
  state13_read_second_client_hello,
  state13_send_server_hello,
  state13_send_server_certificate_verify,
  state13_send_server_finished,
  state13_send_half_rtt_ticket,
  state13_read_second_client_flight,
  state13_process_end_of_early_data,
  state13_read_client_encrypted_extensions,
  state13_read_client_certificate,
  state13_read_client_certificate_verify,
  state13_read_channel_id,
  state13_read_client_finished,
  state13_send_new_session_ticket,
  state13_done,
};

// handback_t lists the points in the state machine where a handback can occur.
// These are the different points at which key material is no longer needed.
enum handback_t {
  handback_after_session_resumption = 0,
  handback_after_ecdhe = 1,
  handback_after_handshake = 2,
  handback_tls13 = 3,
  handback_max_value = handback_tls13,
};

struct SSL_HANDSHAKE {
  explicit SSL_HANDSHAKE(SSL *ssl);
  ~SSL_HANDSHAKE();
  static constexpr bool kAllowUniquePtr = true;

  // ssl is a non-owning pointer to the parent |SSL| object.
  SSL *ssl;

  // config is a non-owning pointer to the handshake configuration.
  SSL_CONFIG *config;

  // wait contains the operation the handshake is currently blocking on or
  // |ssl_hs_ok| if none.
  enum ssl_hs_wait_t wait = ssl_hs_ok;

  // state is the internal state for the TLS 1.2 and below handshake. Its
  // values depend on |do_handshake| but the starting state is always zero.
  int state = 0;

  // tls13_state is the internal state for the TLS 1.3 handshake. Its values
  // depend on |do_handshake| but the starting state is always zero.
  int tls13_state = 0;

  // min_version is the minimum accepted protocol version, taking account both
  // |SSL_OP_NO_*| and |SSL_CTX_set_min_proto_version| APIs.
  uint16_t min_version = 0;

  // max_version is the maximum accepted protocol version, taking account both
  // |SSL_OP_NO_*| and |SSL_CTX_set_max_proto_version| APIs.
  uint16_t max_version = 0;

 private:
  size_t hash_len_ = 0;
  uint8_t secret_[SSL_MAX_MD_SIZE] = {0};
  uint8_t early_traffic_secret_[SSL_MAX_MD_SIZE] = {0};
  uint8_t client_handshake_secret_[SSL_MAX_MD_SIZE] = {0};
  uint8_t server_handshake_secret_[SSL_MAX_MD_SIZE] = {0};
  uint8_t client_traffic_secret_0_[SSL_MAX_MD_SIZE] = {0};
  uint8_t server_traffic_secret_0_[SSL_MAX_MD_SIZE] = {0};
  uint8_t expected_client_finished_[SSL_MAX_MD_SIZE] = {0};

 public:
  void ResizeSecrets(size_t hash_len);

  // GetClientHello, on the server, returns either the normal ClientHello
  // message or the ClientHelloInner if it has been serialized to
  // |ech_client_hello_buf|. This function should only be called when the
  // current message is a ClientHello. It returns true on success and false on
  // error.
  //
  // Note that fields of the returned |out_msg| and |out_client_hello| point
  // into a handshake-owned buffer, so their lifetimes should not exceed this
  // SSL_HANDSHAKE.
  bool GetClientHello(SSLMessage *out_msg, SSL_CLIENT_HELLO *out_client_hello);

  Span<uint8_t> secret() { return MakeSpan(secret_, hash_len_); }
  Span<uint8_t> early_traffic_secret() {
    return MakeSpan(early_traffic_secret_, hash_len_);
  }
  Span<uint8_t> client_handshake_secret() {
    return MakeSpan(client_handshake_secret_, hash_len_);
  }
  Span<uint8_t> server_handshake_secret() {
    return MakeSpan(server_handshake_secret_, hash_len_);
  }
  Span<uint8_t> client_traffic_secret_0() {
    return MakeSpan(client_traffic_secret_0_, hash_len_);
  }
  Span<uint8_t> server_traffic_secret_0() {
    return MakeSpan(server_traffic_secret_0_, hash_len_);
  }
  Span<uint8_t> expected_client_finished() {
    return MakeSpan(expected_client_finished_, hash_len_);
  }

  union {
    // sent is a bitset where the bits correspond to elements of kExtensions
    // in t1_lib.c. Each bit is set if that extension was sent in a
    // ClientHello. It's not used by servers.
    uint32_t sent = 0;
    // received is a bitset, like |sent|, but is used by servers to record
    // which extensions were received from a client.
    uint32_t received;
  } extensions;

  // retry_group is the group ID selected by the server in HelloRetryRequest in
  // TLS 1.3.
  uint16_t retry_group = 0;

  // error, if |wait| is |ssl_hs_error|, is the error the handshake failed on.
  UniquePtr<ERR_SAVE_STATE> error;

  // key_shares are the current key exchange instances. The second is only used
  // as a client if we believe that we should offer two key shares in a
  // ClientHello.
  UniquePtr<SSLKeyShare> key_shares[2];

  // transcript is the current handshake transcript.
  SSLTranscript transcript;

  // cookie is the value of the cookie received from the server, if any.
  Array<uint8_t> cookie;

  // ech_grease contains the bytes of the GREASE ECH extension that was sent in
  // the first ClientHello.
  Array<uint8_t> ech_grease;

  // ech_client_hello_buf, on the server, contains the bytes of the
  // reconstructed ClientHelloInner message.
  Array<uint8_t> ech_client_hello_buf;

  // key_share_bytes is the value of the previously sent KeyShare extension by
  // the client in TLS 1.3.
  Array<uint8_t> key_share_bytes;

  // ecdh_public_key, for servers, is the key share to be sent to the client in
  // TLS 1.3.
  Array<uint8_t> ecdh_public_key;

  // peer_sigalgs are the signature algorithms that the peer supports. These are
  // taken from the contents of the signature algorithms extension for a server
  // or from the CertificateRequest for a client.
  Array<uint16_t> peer_sigalgs;

  // peer_supported_group_list contains the supported group IDs advertised by
  // the peer. This is only set on the server's end. The server does not
  // advertise this extension to the client.
  Array<uint16_t> peer_supported_group_list;

  // peer_delegated_credential_sigalgs are the signature algorithms the peer
  // supports with delegated credentials.
  Array<uint16_t> peer_delegated_credential_sigalgs;

  // peer_key is the peer's ECDH key for a TLS 1.2 client.
  Array<uint8_t> peer_key;

  // negotiated_token_binding_version is used by a server to store the
  // on-the-wire encoding of the Token Binding protocol version to advertise in
  // the ServerHello/EncryptedExtensions if the Token Binding extension is to be
  // sent.
  uint16_t negotiated_token_binding_version;

  // cert_compression_alg_id, for a server, contains the negotiated certificate
  // compression algorithm for this client. It is only valid if
  // |cert_compression_negotiated| is true.
  uint16_t cert_compression_alg_id;

  // ech_hpke_ctx, on the server, is the HPKE context used to decrypt the
  // client's ECH payloads.
  ScopedEVP_HPKE_CTX ech_hpke_ctx;

  // server_params, in a TLS 1.2 server, stores the ServerKeyExchange
  // parameters. It has client and server randoms prepended for signing
  // convenience.
  Array<uint8_t> server_params;

  // peer_psk_identity_hint, on the client, is the psk_identity_hint sent by the
  // server when using a TLS 1.2 PSK key exchange.
  UniquePtr<char> peer_psk_identity_hint;

  // ca_names, on the client, contains the list of CAs received in a
  // CertificateRequest message.
  UniquePtr<STACK_OF(CRYPTO_BUFFER)> ca_names;

  // cached_x509_ca_names contains a cache of parsed versions of the elements of
  // |ca_names|. This pointer is left non-owning so only
  // |ssl_crypto_x509_method| needs to link against crypto/x509.
  STACK_OF(X509_NAME) *cached_x509_ca_names = nullptr;

  // certificate_types, on the client, contains the set of certificate types
  // received in a CertificateRequest message.
  Array<uint8_t> certificate_types;

  // local_pubkey is the public key we are authenticating as.
  UniquePtr<EVP_PKEY> local_pubkey;

  // peer_pubkey is the public key parsed from the peer's leaf certificate.
  UniquePtr<EVP_PKEY> peer_pubkey;

  // new_session is the new mutable session being established by the current
  // handshake. It should not be cached.
  UniquePtr<SSL_SESSION> new_session;

  // early_session is the session corresponding to the current 0-RTT state on
  // the client if |in_early_data| is true.
  UniquePtr<SSL_SESSION> early_session;

  // ech_server_config_list, for servers, is the list of ECHConfig values that
  // were valid when the server received the first ClientHello. Its value will
  // not change when the config list on |SSL_CTX| is updated.
  UniquePtr<SSL_ECH_SERVER_CONFIG_LIST> ech_server_config_list;

  // new_cipher is the cipher being negotiated in this handshake.
  const SSL_CIPHER *new_cipher = nullptr;

  // key_block is the record-layer key block for TLS 1.2 and earlier.
  Array<uint8_t> key_block;

  // ech_accept, on the server, indicates whether the server should overwrite
  // part of ServerHello.random with the ECH accept_confirmation value.
  bool ech_accept : 1;

  // ech_present, on the server, indicates whether the ClientHello contained an
  // encrypted_client_hello extension.
  bool ech_present : 1;

  // ech_is_inner_present, on the server, indicates whether the ClientHello
  // contained an ech_is_inner extension.
  bool ech_is_inner_present : 1;

  // scts_requested is true if the SCT extension is in the ClientHello.
  bool scts_requested : 1;

  // needs_psk_binder is true if the ClientHello has a placeholder PSK binder to
  // be filled in.
  bool needs_psk_binder : 1;

  // handshake_finalized is true once the handshake has completed, at which
  // point accessors should use the established state.
  bool handshake_finalized : 1;

  // accept_psk_mode stores whether the client's PSK mode is compatible with our
  // preferences.
  bool accept_psk_mode : 1;

  // cert_request is true if a client certificate was requested.
  bool cert_request : 1;

  // certificate_status_expected is true if OCSP stapling was negotiated and the
  // server is expected to send a CertificateStatus message. (This is used on
  // both the client and server sides.)
  bool certificate_status_expected : 1;

  // ocsp_stapling_requested is true if a client requested OCSP stapling.
  bool ocsp_stapling_requested : 1;

  // delegated_credential_requested is true if the peer indicated support for
  // the delegated credential extension.
  bool delegated_credential_requested : 1;

  // should_ack_sni is used by a server and indicates that the SNI extension
  // should be echoed in the ServerHello.
  bool should_ack_sni : 1;

  // in_false_start is true if there is a pending client handshake in False
  // Start. The client may write data at this point.
  bool in_false_start : 1;

  // in_early_data is true if there is a pending handshake that has progressed
  // enough to send and receive early data.
  bool in_early_data : 1;

  // early_data_offered is true if the client sent the early_data extension.
  bool early_data_offered : 1;

  // can_early_read is true if application data may be read at this point in the
  // handshake.
  bool can_early_read : 1;

  // can_early_write is true if application data may be written at this point in
  // the handshake.
  bool can_early_write : 1;

  // next_proto_neg_seen is one of NPN was negotiated.
  bool next_proto_neg_seen : 1;

  // ticket_expected is true if a TLS 1.2 NewSessionTicket message is to be sent
  // or received.
  bool ticket_expected : 1;

  // extended_master_secret is true if the extended master secret extension is
  // negotiated in this handshake.
  bool extended_master_secret : 1;

  // pending_private_key_op is true if there is a pending private key operation
  // in progress.
  bool pending_private_key_op : 1;

  // grease_seeded is true if |grease_seed| has been initialized.
  bool grease_seeded : 1;

  // handback indicates that a server should pause the handshake after
  // finishing operations that require private key material, in such a way that
  // |SSL_get_error| returns |SSL_ERROR_HANDBACK|.  It is set by
  // |SSL_apply_handoff|.
  bool handback : 1;

  // cert_compression_negotiated is true iff |cert_compression_alg_id| is valid.
  bool cert_compression_negotiated : 1;

  // apply_jdk11_workaround is true if the peer is probably a JDK 11 client
  // which implemented TLS 1.3 incorrectly.
  bool apply_jdk11_workaround : 1;

  // client_version is the value sent or received in the ClientHello version.
  uint16_t client_version = 0;

  // early_data_read is the amount of early data that has been read by the
  // record layer.
  uint16_t early_data_read = 0;

  // early_data_written is the amount of early data that has been written by the
  // record layer.
  uint16_t early_data_written = 0;

  // session_id is the session ID in the ClientHello.
  uint8_t session_id[SSL_MAX_SSL_SESSION_ID_LENGTH] = {0};
  uint8_t session_id_len = 0;

  // grease_seed is the entropy for GREASE values. It is valid if
  // |grease_seeded| is true.
  uint8_t grease_seed[ssl_grease_last_index + 1] = {0};
};

```

#### 1.2.2 SSL握手模板结构

握手模板结构是原始握手的模板，通过跟这个协商产生具体的握手协议。 `SSL_CTX` 结构是线程安全的。

```c++
struct ssl_ctx_st {
  explicit ssl_ctx_st(const SSL_METHOD *ssl_method);
  ssl_ctx_st(const ssl_ctx_st &) = delete;
  ssl_ctx_st &operator=(const ssl_ctx_st &) = delete;

  const bssl::SSL_PROTOCOL_METHOD *method = nullptr;
  const bssl::SSL_X509_METHOD *x509_method = nullptr;

  // lock is used to protect various operations on this object.
  CRYPTO_MUTEX lock;

  // conf_max_version is the maximum acceptable protocol version configured by
  // |SSL_CTX_set_max_proto_version|. Note this version is normalized in DTLS
  // and is further constrainted by |SSL_OP_NO_*|.
  uint16_t conf_max_version = 0;

  // conf_min_version is the minimum acceptable protocol version configured by
  // |SSL_CTX_set_min_proto_version|. Note this version is normalized in DTLS
  // and is further constrainted by |SSL_OP_NO_*|.
  uint16_t conf_min_version = 0;

  // quic_method is the method table corresponding to the QUIC hooks.
  const SSL_QUIC_METHOD *quic_method = nullptr;

  bssl::UniquePtr<bssl::SSLCipherPreferenceList> cipher_list;

  X509_STORE *cert_store = nullptr;
  LHASH_OF(SSL_SESSION) *sessions = nullptr;
  // Most session-ids that will be cached, default is
  // SSL_SESSION_CACHE_MAX_SIZE_DEFAULT. 0 is unlimited.
  unsigned long session_cache_size = SSL_SESSION_CACHE_MAX_SIZE_DEFAULT;
  SSL_SESSION *session_cache_head = nullptr;
  SSL_SESSION *session_cache_tail = nullptr;

  // handshakes_since_cache_flush is the number of successful handshakes since
  // the last cache flush.
  int handshakes_since_cache_flush = 0;

  // This can have one of 2 values, ored together,
  // SSL_SESS_CACHE_CLIENT,
  // SSL_SESS_CACHE_SERVER,
  // Default is SSL_SESSION_CACHE_SERVER, which means only
  // SSL_accept which cache SSL_SESSIONS.
  int session_cache_mode = SSL_SESS_CACHE_SERVER;

  // session_timeout is the default lifetime for new sessions in TLS 1.2 and
  // earlier, in seconds.
  uint32_t session_timeout = SSL_DEFAULT_SESSION_TIMEOUT;

  // session_psk_dhe_timeout is the default lifetime for new sessions in TLS
  // 1.3, in seconds.
  uint32_t session_psk_dhe_timeout = SSL_DEFAULT_SESSION_PSK_DHE_TIMEOUT;

  // If this callback is not null, it will be called each time a session id is
  // added to the cache.  If this function returns 1, it means that the
  // callback will do a SSL_SESSION_free() when it has finished using it.
  // Otherwise, on 0, it means the callback has finished with it. If
  // remove_session_cb is not null, it will be called when a session-id is
  // removed from the cache.  After the call, OpenSSL will SSL_SESSION_free()
  // it.
  int (*new_session_cb)(SSL *ssl, SSL_SESSION *sess) = nullptr;
  void (*remove_session_cb)(SSL_CTX *ctx, SSL_SESSION *sess) = nullptr;
  SSL_SESSION *(*get_session_cb)(SSL *ssl, const uint8_t *data, int len,
                                 int *copy) = nullptr;

  CRYPTO_refcount_t references = 1;

  // if defined, these override the X509_verify_cert() calls
  int (*app_verify_callback)(X509_STORE_CTX *store_ctx, void *arg) = nullptr;
  void *app_verify_arg = nullptr;

  ssl_verify_result_t (*custom_verify_callback)(SSL *ssl,
                                                uint8_t *out_alert) = nullptr;

  // Default password callback.
  pem_password_cb *default_passwd_callback = nullptr;

  // Default password callback user data.
  void *default_passwd_callback_userdata = nullptr;

  // get client cert callback
  int (*client_cert_cb)(SSL *ssl, X509 **out_x509,
                        EVP_PKEY **out_pkey) = nullptr;

  // get channel id callback
  void (*channel_id_cb)(SSL *ssl, EVP_PKEY **out_pkey) = nullptr;

  CRYPTO_EX_DATA ex_data;

  // Default values used when no per-SSL value is defined follow

  void (*info_callback)(const SSL *ssl, int type, int value) = nullptr;

  // what we put in client cert requests
  bssl::UniquePtr<STACK_OF(CRYPTO_BUFFER)> client_CA;

  // cached_x509_client_CA is a cache of parsed versions of the elements of
  // |client_CA|.
  STACK_OF(X509_NAME) *cached_x509_client_CA = nullptr;


  // Default values to use in SSL structures follow (these are copied by
  // SSL_new)

  uint32_t options = 0;
  // Disable the auto-chaining feature by default. wpa_supplicant relies on this
  // feature, but require callers opt into it.
  uint32_t mode = SSL_MODE_NO_AUTO_CHAIN;
  uint32_t max_cert_list = SSL_MAX_CERT_LIST_DEFAULT;

  bssl::UniquePtr<bssl::CERT> cert;

  // callback that allows applications to peek at protocol messages
  void (*msg_callback)(int write_p, int version, int content_type,
                       const void *buf, size_t len, SSL *ssl,
                       void *arg) = nullptr;
  void *msg_callback_arg = nullptr;

  int verify_mode = SSL_VERIFY_NONE;
  int (*default_verify_callback)(int ok, X509_STORE_CTX *ctx) =
      nullptr;  // called 'verify_callback' in the SSL

  X509_VERIFY_PARAM *param = nullptr;

  // select_certificate_cb is called before most ClientHello processing and
  // before the decision whether to resume a session is made. See
  // |ssl_select_cert_result_t| for details of the return values.
  ssl_select_cert_result_t (*select_certificate_cb)(const SSL_CLIENT_HELLO *) =
      nullptr;

  // dos_protection_cb is called once the resumption decision for a ClientHello
  // has been made. It returns one to continue the handshake or zero to
  // abort.
  int (*dos_protection_cb)(const SSL_CLIENT_HELLO *) = nullptr;

  // Controls whether to verify certificates when resuming connections. They
  // were already verified when the connection was first made, so the default is
  // false. For now, this is only respected on clients, not servers.
  bool reverify_on_resume = false;

  // Maximum amount of data to send in one fragment. actual record size can be
  // more than this due to padding and MAC overheads.
  uint16_t max_send_fragment = SSL3_RT_MAX_PLAIN_LENGTH;

  // TLS extensions servername callback
  int (*servername_callback)(SSL *, int *, void *) = nullptr;
  void *servername_arg = nullptr;

  // RFC 4507 session ticket keys. |ticket_key_current| may be NULL before the
  // first handshake and |ticket_key_prev| may be NULL at any time.
  // Automatically generated ticket keys are rotated as needed at handshake
  // time. Hence, all access must be synchronized through |lock|.
  bssl::UniquePtr<bssl::TicketKey> ticket_key_current;
  bssl::UniquePtr<bssl::TicketKey> ticket_key_prev;

  // Callback to support customisation of ticket key setting
  int (*ticket_key_cb)(SSL *ssl, uint8_t *name, uint8_t *iv,
                       EVP_CIPHER_CTX *ectx, HMAC_CTX *hctx, int enc) = nullptr;

  // Server-only: psk_identity_hint is the default identity hint to send in
  // PSK-based key exchanges.
  bssl::UniquePtr<char> psk_identity_hint;

  unsigned (*psk_client_callback)(SSL *ssl, const char *hint, char *identity,
                                  unsigned max_identity_len, uint8_t *psk,
                                  unsigned max_psk_len) = nullptr;
  unsigned (*psk_server_callback)(SSL *ssl, const char *identity, uint8_t *psk,
                                  unsigned max_psk_len) = nullptr;


  // Next protocol negotiation information
  // (for experimental NPN extension).

  // For a server, this contains a callback function by which the set of
  // advertised protocols can be provided.
  int (*next_protos_advertised_cb)(SSL *ssl, const uint8_t **out,
                                   unsigned *out_len, void *arg) = nullptr;
  void *next_protos_advertised_cb_arg = nullptr;
  // For a client, this contains a callback function that selects the
  // next protocol from the list provided by the server.
  int (*next_proto_select_cb)(SSL *ssl, uint8_t **out, uint8_t *out_len,
                              const uint8_t *in, unsigned in_len,
                              void *arg) = nullptr;
  void *next_proto_select_cb_arg = nullptr;

  // ALPN information
  // (we are in the process of transitioning from NPN to ALPN.)

  // For a server, this contains a callback function that allows the
  // server to select the protocol for the connection.
  //   out: on successful return, this must point to the raw protocol
  //        name (without the length prefix).
  //   outlen: on successful return, this contains the length of |*out|.
  //   in: points to the client's list of supported protocols in
  //       wire-format.
  //   inlen: the length of |in|.
  int (*alpn_select_cb)(SSL *ssl, const uint8_t **out, uint8_t *out_len,
                        const uint8_t *in, unsigned in_len,
                        void *arg) = nullptr;
  void *alpn_select_cb_arg = nullptr;

  // For a client, this contains the list of supported protocols in wire
  // format.
  bssl::Array<uint8_t> alpn_client_proto_list;

  // SRTP profiles we are willing to do from RFC 5764
  bssl::UniquePtr<STACK_OF(SRTP_PROTECTION_PROFILE)> srtp_profiles;

  // Defined compression algorithms for certificates.
  bssl::GrowableArray<bssl::CertCompressionAlg> cert_compression_algs;

  // Supported group values inherited by SSL structure
  bssl::Array<uint16_t> supported_group_list;

  // The client's Channel ID private key.
  bssl::UniquePtr<EVP_PKEY> channel_id_private;

  // ech_server_config_list contains the server's list of ECHConfig values and
  // associated private keys. This list may be swapped out at any time, so all
  // access must be synchronized through |lock|.
  bssl::UniquePtr<SSL_ECH_SERVER_CONFIG_LIST> ech_server_config_list;

  // keylog_callback, if not NULL, is the key logging callback. See
  // |SSL_CTX_set_keylog_callback|.
  void (*keylog_callback)(const SSL *ssl, const char *line) = nullptr;

  // current_time_cb, if not NULL, is the function to use to get the current
  // time. It sets |*out_clock| to the current time. The |ssl| argument is
  // always NULL. See |SSL_CTX_set_current_time_cb|.
  void (*current_time_cb)(const SSL *ssl, struct timeval *out_clock) = nullptr;

  // pool is used for all |CRYPTO_BUFFER|s in case we wish to share certificate
  // memory.
  CRYPTO_BUFFER_POOL *pool = nullptr;

  // ticket_aead_method contains function pointers for opening and sealing
  // session tickets.
  const SSL_TICKET_AEAD_METHOD *ticket_aead_method = nullptr;

  // legacy_ocsp_callback implements an OCSP-related callback for OpenSSL
  // compatibility.
  int (*legacy_ocsp_callback)(SSL *ssl, void *arg) = nullptr;
  void *legacy_ocsp_callback_arg = nullptr;

  // verify_sigalgs, if not empty, is the set of signature algorithms
  // accepted from the peer in decreasing order of preference.
  bssl::Array<uint16_t> verify_sigalgs;

  // retain_only_sha256_of_client_certs is true if we should compute the SHA256
  // hash of the peer's certificate and then discard it to save memory and
  // session space. Only effective on the server side.
  bool retain_only_sha256_of_client_certs : 1;

  // quiet_shutdown is true if the connection should not send a close_notify on
  // shutdown.
  bool quiet_shutdown : 1;

  // ocsp_stapling_enabled is only used by client connections and indicates
  // whether OCSP stapling will be requested.
  bool ocsp_stapling_enabled : 1;

  // If true, a client will request certificate timestamps.
  bool signed_cert_timestamps_enabled : 1;

  // channel_id_enabled is whether Channel ID is enabled. For a server, means
  // that we'll accept Channel IDs from clients.  For a client, means that we'll
  // advertise support.
  bool channel_id_enabled : 1;

  // grease_enabled is whether draft-davidben-tls-grease-01 is enabled.
  bool grease_enabled : 1;

  // allow_unknown_alpn_protos is whether the client allows unsolicited ALPN
  // protocols from the peer.
  bool allow_unknown_alpn_protos : 1;

  // false_start_allowed_without_alpn is whether False Start (if
  // |SSL_MODE_ENABLE_FALSE_START| is enabled) is allowed without ALPN.
  bool false_start_allowed_without_alpn : 1;

  // handoff indicates that a server should stop after receiving the
  // ClientHello and pause the handshake in such a way that |SSL_get_error|
  // returns |SSL_ERROR_HANDOFF|.
  bool handoff : 1;

  // If enable_early_data is true, early data can be sent and accepted.
  bool enable_early_data : 1;

 private:
  ~ssl_ctx_st();
  friend void SSL_CTX_free(SSL_CTX *);
};
```



### 1.3 握手/加解密过程当中涉及到的数据结构

#### 1.3.1 transcript hash数据结构

transcript实际上就是握手报文的完整结构，直接翻译为抄本，除了TLS1.3的HELLO RETRY REQUEST会加上一些特殊的地方，其他都是由报文拼接。可以看到，GOOGLE把所有的跟HASH相关的动作都包裹到`SSLTranscript`结构中。但是从功能的角度来看，我觉得这个就属于C++的强行面向对象了，我个人很厌烦这一点，不像C那么灵活和彻底。

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



#### 1.3.2 AEAD context

严格来说TLS1.2就有AEAD，不过在TLS1.3中正式大规模使用，具体的

```c++
class SSLAEADContext {
 public:
  SSLAEADContext(uint16_t version, bool is_dtls, const SSL_CIPHER *cipher);
  ~SSLAEADContext();
  static constexpr bool kAllowUniquePtr = true;

  SSLAEADContext(const SSLAEADContext &&) = delete;
  SSLAEADContext &operator=(const SSLAEADContext &&) = delete;

  // CreateNullCipher creates an |SSLAEADContext| for the null cipher.
  static UniquePtr<SSLAEADContext> CreateNullCipher(bool is_dtls);

  // Create creates an |SSLAEADContext| using the supplied key material. It
  // returns nullptr on error. Only one of |Open| or |Seal| may be used with the
  // resulting object, depending on |direction|. |version| is the normalized
  // protocol version, so DTLS 1.0 is represented as 0x0301, not 0xffef.
  static UniquePtr<SSLAEADContext> Create(enum evp_aead_direction_t direction,
                                          uint16_t version, bool is_dtls,
                                          const SSL_CIPHER *cipher,
                                          Span<const uint8_t> enc_key,
                                          Span<const uint8_t> mac_key,
                                          Span<const uint8_t> fixed_iv);

  // CreatePlaceholderForQUIC creates a placeholder |SSLAEADContext| for the
  // given cipher and version. The resulting object can be queried for various
  // properties but cannot encrypt or decrypt data.
  static UniquePtr<SSLAEADContext> CreatePlaceholderForQUIC(
      uint16_t version, const SSL_CIPHER *cipher);

  // SetVersionIfNullCipher sets the version the SSLAEADContext for the null
  // cipher, to make version-specific determinations in the record layer prior
  // to a cipher being selected.
  void SetVersionIfNullCipher(uint16_t version);

  // ProtocolVersion returns the protocol version associated with this
  // SSLAEADContext. It can only be called once |version_| has been set to a
  // valid value.
  uint16_t ProtocolVersion() const;

  // RecordVersion returns the record version that should be used with this
  // SSLAEADContext for record construction and crypto.
  uint16_t RecordVersion() const;

  const SSL_CIPHER *cipher() const { return cipher_; }

  // is_null_cipher returns true if this is the null cipher.
  bool is_null_cipher() const { return !cipher_; }

  // ExplicitNonceLen returns the length of the explicit nonce.
  size_t ExplicitNonceLen() const;

  // MaxOverhead returns the maximum overhead of calling |Seal|.
  size_t MaxOverhead() const;

  // SuffixLen calculates the suffix length written by |SealScatter| and writes
  // it to |*out_suffix_len|. It returns true on success and false on error.
  // |in_len| and |extra_in_len| should equal the argument of the same names
  // passed to |SealScatter|.
  bool SuffixLen(size_t *out_suffix_len, size_t in_len,
                 size_t extra_in_len) const;

  // CiphertextLen calculates the total ciphertext length written by
  // |SealScatter| and writes it to |*out_len|. It returns true on success and
  // false on error. |in_len| and |extra_in_len| should equal the argument of
  // the same names passed to |SealScatter|.
  bool CiphertextLen(size_t *out_len, size_t in_len, size_t extra_in_len) const;

  // Open authenticates and decrypts |in| in-place. On success, it sets |*out|
  // to the plaintext in |in| and returns true.  Otherwise, it returns
  // false. The output will always be |ExplicitNonceLen| bytes ahead of |in|.
  bool Open(Span<uint8_t> *out, uint8_t type, uint16_t record_version,
            const uint8_t seqnum[8], Span<const uint8_t> header,
            Span<uint8_t> in);

  // Seal encrypts and authenticates |in_len| bytes from |in| and writes the
  // result to |out|. It returns true on success and false on error.
  //
  // If |in| and |out| alias then |out| + |ExplicitNonceLen| must be == |in|.
  bool Seal(uint8_t *out, size_t *out_len, size_t max_out, uint8_t type,
            uint16_t record_version, const uint8_t seqnum[8],
            Span<const uint8_t> header, const uint8_t *in, size_t in_len);

  // SealScatter encrypts and authenticates |in_len| bytes from |in| and splits
  // the result between |out_prefix|, |out| and |out_suffix|. It returns one on
  // success and zero on error.
  //
  // On successful return, exactly |ExplicitNonceLen| bytes are written to
  // |out_prefix|, |in_len| bytes to |out|, and |SuffixLen| bytes to
  // |out_suffix|.
  //
  // |extra_in| may point to an additional plaintext buffer. If present,
  // |extra_in_len| additional bytes are encrypted and authenticated, and the
  // ciphertext is written to the beginning of |out_suffix|. |SuffixLen| should
  // be used to size |out_suffix| accordingly.
  //
  // If |in| and |out| alias then |out| must be == |in|. Other arguments may not
  // alias anything.
  bool SealScatter(uint8_t *out_prefix, uint8_t *out, uint8_t *out_suffix,
                   uint8_t type, uint16_t record_version,
                   const uint8_t seqnum[8], Span<const uint8_t> header,
                   const uint8_t *in, size_t in_len, const uint8_t *extra_in,
                   size_t extra_in_len);

  bool GetIV(const uint8_t **out_iv, size_t *out_iv_len) const;

 private:
  // GetAdditionalData returns the additional data, writing into |storage| if
  // necessary.
  Span<const uint8_t> GetAdditionalData(uint8_t storage[13], uint8_t type,
                                        uint16_t record_version,
                                        const uint8_t seqnum[8],
                                        size_t plaintext_len,
                                        Span<const uint8_t> header);

  const SSL_CIPHER *cipher_;
  ScopedEVP_AEAD_CTX ctx_;
  // fixed_nonce_ contains any bytes of the nonce that are fixed for all
  // records.
  uint8_t fixed_nonce_[12];
  uint8_t fixed_nonce_len_ = 0, variable_nonce_len_ = 0;
  // version_ is the wire version that should be used with this AEAD.
  uint16_t version_;
  // is_dtls_ is whether DTLS is being used with this AEAD.
  bool is_dtls_;
  // variable_nonce_included_in_record_ is true if the variable nonce
  // for a record is included as a prefix before the ciphertext.
  bool variable_nonce_included_in_record_ : 1;
  // random_variable_nonce_ is true if the variable nonce is
  // randomly generated, rather than derived from the sequence
  // number.
  bool random_variable_nonce_ : 1;
  // xor_fixed_nonce_ is true if the fixed nonce should be XOR'd into the
  // variable nonce rather than prepended.
  bool xor_fixed_nonce_ : 1;
  // omit_length_in_ad_ is true if the length should be omitted in the
  // AEAD's ad parameter.
  bool omit_length_in_ad_ : 1;
  // ad_is_header_ is true if the AEAD's ad parameter is the record header.
  bool ad_is_header_ : 1;
};

```



#### 1.3.3 Key Share结构



```c++
class SSLKeyShare {
 public:
  virtual ~SSLKeyShare() {}
  static constexpr bool kAllowUniquePtr = true;
  HAS_VIRTUAL_DESTRUCTOR

  // Create returns a SSLKeyShare instance for use with group |group_id| or
  // nullptr on error.
  static UniquePtr<SSLKeyShare> Create(uint16_t group_id);

  // Create deserializes an SSLKeyShare instance previously serialized by
  // |Serialize|.
  static UniquePtr<SSLKeyShare> Create(CBS *in);

  // Serializes writes the group ID and private key, in a format that can be
  // read by |Create|.
  bool Serialize(CBB *out);

  // GroupID returns the group ID.
  virtual uint16_t GroupID() const PURE_VIRTUAL;

  // Offer generates a keypair and writes the public value to
  // |out_public_key|. It returns true on success and false on error.
  virtual bool Offer(CBB *out_public_key) PURE_VIRTUAL;

  // Accept performs a key exchange against the |peer_key| generated by |Offer|.
  // On success, it returns true, writes the public value to |out_public_key|,
  // and sets |*out_secret| to the shared secret. On failure, it returns false
  // and sets |*out_alert| to an alert to send to the peer.
  //
  // The default implementation calls |Offer| and then |Finish|, assuming a key
  // exchange protocol where the peers are symmetric.
  virtual bool Accept(CBB *out_public_key, Array<uint8_t> *out_secret,
                      uint8_t *out_alert, Span<const uint8_t> peer_key);

  // Finish performs a key exchange against the |peer_key| generated by
  // |Accept|. On success, it returns true and sets |*out_secret| to the shared
  // secret. On failure, it returns false and sets |*out_alert| to an alert to
  // send to the peer.
  virtual bool Finish(Array<uint8_t> *out_secret, uint8_t *out_alert,
                      Span<const uint8_t> peer_key) PURE_VIRTUAL;

  // SerializePrivateKey writes the private key to |out|, returning true if
  // successful and false otherwise. It should be called after |Offer|.
  virtual bool SerializePrivateKey(CBB *out) { return false; }

  // DeserializePrivateKey initializes the state of the key exchange from |in|,
  // returning true if successful and false otherwise.
  virtual bool DeserializePrivateKey(CBS *in) { return false; }
};

```



#### 1.3.4 ECH 拓展

没想到BoringSSL已经支持ECH了，

```c++
class ECHServerConfig {
 public:
  ECHServerConfig() : is_retry_config_(false), initialized_(false) {}
  ECHServerConfig(ECHServerConfig &&other) = default;
  ~ECHServerConfig() = default;
  ECHServerConfig &operator=(ECHServerConfig &&) = default;

  // Init parses |ech_config| as an ECHConfig and saves a copy of |private_key|.
  // It returns true on success and false on error. It will also error if
  // |private_key| is not a valid X25519 private key or it does not correspond
  // to the parsed public key.
  bool Init(Span<const uint8_t> ech_config, Span<const uint8_t> private_key,
            bool is_retry_config);

  // SupportsCipherSuite returns true when this ECHConfig supports the HPKE
  // ciphersuite composed of |kdf_id| and |aead_id|. This function must only be
  // called on an initialized object.
  bool SupportsCipherSuite(uint16_t kdf_id, uint16_t aead_id) const;

  Span<const uint8_t> raw() const {
    assert(initialized_);
    return raw_;
  }
  Span<const uint8_t> public_key() const {
    assert(initialized_);
    return public_key_;
  }
  Span<const uint8_t> private_key() const {
    assert(initialized_);
    return MakeConstSpan(private_key_, sizeof(private_key_));
  }
  Span<const uint8_t> config_id_sha256() const {
    assert(initialized_);
    return MakeConstSpan(config_id_sha256_, sizeof(config_id_sha256_));
  }
  bool is_retry_config() const {
    assert(initialized_);
    return is_retry_config_;
  }

 private:
  Array<uint8_t> raw_;
  Span<const uint8_t> public_key_;
  Span<const uint8_t> cipher_suites_;

  // private_key_ is the key corresponding to |public_key|. For clients, it must
  // be empty (|private_key_present_ == false|). For servers, it must be a valid
  // X25519 private key.
  uint8_t private_key_[X25519_PRIVATE_KEY_LEN];

  // config_id_ stores the precomputed result of |ConfigID| for
  // |EVP_HPKE_HKDF_SHA256|.
  uint8_t config_id_sha256_[8];

  bool is_retry_config_ : 1;
  bool initialized_ : 1;
};
```



#### 1.3.5 SSL_CONFIG结构

SSL_CONFIG用于存储一些握手完成之后需要的数据结构，

```c++
struct SSL_CONFIG {
  static constexpr bool kAllowUniquePtr = true;

  explicit SSL_CONFIG(SSL *ssl_arg);
  ~SSL_CONFIG();

  // ssl is a non-owning pointer to the parent |SSL| object.
  SSL *const ssl = nullptr;

  // conf_max_version is the maximum acceptable version configured by
  // |SSL_set_max_proto_version|. Note this version is not normalized in DTLS
  // and is further constrained by |SSL_OP_NO_*|.
  uint16_t conf_max_version = 0;

  // conf_min_version is the minimum acceptable version configured by
  // |SSL_set_min_proto_version|. Note this version is not normalized in DTLS
  // and is further constrained by |SSL_OP_NO_*|.
  uint16_t conf_min_version = 0;

  X509_VERIFY_PARAM *param = nullptr;

  // crypto
  UniquePtr<SSLCipherPreferenceList> cipher_list;

  // This is used to hold the local certificate used (i.e. the server
  // certificate for a server or the client certificate for a client).
  UniquePtr<CERT> cert;

  int (*verify_callback)(int ok,
                         X509_STORE_CTX *ctx) =
      nullptr;  // fail if callback returns 0

  enum ssl_verify_result_t (*custom_verify_callback)(
      SSL *ssl, uint8_t *out_alert) = nullptr;
  // Server-only: psk_identity_hint is the identity hint to send in
  // PSK-based key exchanges.
  UniquePtr<char> psk_identity_hint;

  unsigned (*psk_client_callback)(SSL *ssl, const char *hint, char *identity,
                                  unsigned max_identity_len, uint8_t *psk,
                                  unsigned max_psk_len) = nullptr;
  unsigned (*psk_server_callback)(SSL *ssl, const char *identity, uint8_t *psk,
                                  unsigned max_psk_len) = nullptr;

  // for server side, keep the list of CA_dn we can use
  UniquePtr<STACK_OF(CRYPTO_BUFFER)> client_CA;

  // cached_x509_client_CA is a cache of parsed versions of the elements of
  // |client_CA|.
  STACK_OF(X509_NAME) *cached_x509_client_CA = nullptr;

  Array<uint16_t> supported_group_list;  // our list

  // The client's Channel ID private key.
  UniquePtr<EVP_PKEY> channel_id_private;

  // For a client, this contains the list of supported protocols in wire
  // format.
  Array<uint8_t> alpn_client_proto_list;

  // alps_configs contains the list of supported protocols to use with ALPS,
  // along with their corresponding ALPS values.
  GrowableArray<ALPSConfig> alps_configs;

  // Contains a list of supported Token Binding key parameters.
  Array<uint8_t> token_binding_params;

  // Contains the QUIC transport params that this endpoint will send.
  Array<uint8_t> quic_transport_params;

  // Contains the context used to decide whether to accept early data in QUIC.
  Array<uint8_t> quic_early_data_context;

  // verify_sigalgs, if not empty, is the set of signature algorithms
  // accepted from the peer in decreasing order of preference.
  Array<uint16_t> verify_sigalgs;

  // srtp_profiles is the list of configured SRTP protection profiles for
  // DTLS-SRTP.
  UniquePtr<STACK_OF(SRTP_PROTECTION_PROFILE)> srtp_profiles;

  // verify_mode is a bitmask of |SSL_VERIFY_*| values.
  uint8_t verify_mode = SSL_VERIFY_NONE;

  // ech_grease_enabled controls whether ECH GREASE may be sent in the
  // ClientHello.
  bool ech_grease_enabled : 1;

  // Enable signed certificate time stamps. Currently client only.
  bool signed_cert_timestamps_enabled : 1;

  // ocsp_stapling_enabled is only used by client connections and indicates
  // whether OCSP stapling will be requested.
  bool ocsp_stapling_enabled : 1;

  // channel_id_enabled is copied from the |SSL_CTX|. For a server, means that
  // we'll accept Channel IDs from clients. For a client, means that we'll
  // advertise support.
  bool channel_id_enabled : 1;

  // If enforce_rsa_key_usage is true, the handshake will fail if the
  // keyUsage extension is present and incompatible with the TLS usage.
  // This field is not read until after certificate verification.
  bool enforce_rsa_key_usage : 1;

  // retain_only_sha256_of_client_certs is true if we should compute the SHA256
  // hash of the peer's certificate and then discard it to save memory and
  // session space. Only effective on the server side.
  bool retain_only_sha256_of_client_certs : 1;

  // handoff indicates that a server should stop after receiving the
  // ClientHello and pause the handshake in such a way that |SSL_get_error|
  // returns |SSL_ERROR_HANDOFF|. This is copied in |SSL_new| from the |SSL_CTX|
  // element of the same name and may be cleared if the handoff is declined.
  bool handoff : 1;

  // shed_handshake_config indicates that the handshake config (this object!)
  // should be freed after the handshake completes.
  bool shed_handshake_config : 1;

  // jdk11_workaround is whether to disable TLS 1.3 for JDK 11 clients, as a
  // workaround for https://bugs.openjdk.java.net/browse/JDK-8211806.
  bool jdk11_workaround : 1;

  // QUIC drafts up to and including 32 used a different TLS extension
  // codepoint to convey QUIC's transport parameters.
  bool quic_use_legacy_codepoint : 1;
};
```

## 2 握手函数

### 2.1 握手相关函数

函数`SSL_do_handshake`负责继续当前连接的握手状态，有个很有趣的地方实际上要注意，需要先绑定`fd`或`bio`才能继续让`SSL_do_handshake`继续。很类似的`SSL__connect`和`SSL_accept`分别用来给client和server端使用，`SSL_read`函数和openssl代码里还是一致的，返回读取到的明文。和`SSL_read`类似的函数`SSL_peak`用来获得明文，但是不会真正的读取报文。这东西也就特定情况下需要。



诸如`KEY_UPDATE_UPDATE_REQUESTED`和`KEY_UPDATE_UPDATE_NOT_REQUESTED`倒是和我写的代码完全一致。



`SSL_get_error`函数很有趣，最近的操作失败时，使用这个函数获取失败原因，统一的处理



## 3 基本的属性和值

### 3.1 protocol version和函数

和openssl一样，还是两个函数。但是有单独的函数去设定对应的SSL版本号。最后的函数负责获得握手时确定的版本号

```c++
#define DTLS1_VERSION_MAJOR 0xfe
#define SSL3_VERSION_MAJOR 0x03
#define SSL3_VERSION 0x0300
#define TLS1_VERSION 0x0301
#define TLS1_1_VERSION 0x0302
#define TLS1_2_VERSION 0x0303
#define TLS1_3_VERSION 0x0304
#define DTLS1_VERSION 0xfeff
#define DTLS1_2_VERSION 0xfefd
OPENSSL_EXPORT int SSL_CTX_set_min_proto_version(SSL_CTX *ctx, uint16_t version);
OPENSSL_EXPORT int SSL_CTX_set_max_proto_version(SSL_CTX *ctx, uint16_t version);                           OPENSSL_EXPORT int SSL_version(const SSL *ssl);                      
```



### 3.2 CTX的属性和函数

属性这一块坦白讲我觉得没啥意思，

```
SSL_MODE_ENABLE_PARTIAL_WRITE //允许往一个record里面写部分的报文
```

### 3.3 证书和私钥相关函数

```c++
OPENSSL_EXPORT int SSL_CTX_use_certificate(SSL_CTX *ctx, X509 *x509);
OPENSSL_EXPORT int SSL_use_certificate(SSL *ssl, X509 *x509);
OPENSSL_EXPORT int SSL_CTX_set0_chain(SSL_CTX *ctx, STACK_OF(X509) *chain);
OPENSSL_EXPORT int SSL_CTX_set1_chain(SSL_CTX *ctx, STACK_OF(X509) *chain);

OPENSSL_EXPORT void SSL_CTX_set_cert_cb(SSL_CTX *ctx,
                                        int (*cb)(SSL *ssl, void *arg),
                                        void *arg);  //这个函数就很有趣了，设置回调来选择具体的证书
OPENSSL_EXPORT size_t
SSL_get0_peer_verify_algorithms(const SSL *ssl, const uint16_t **out_sigalgs);//这个函数就负责选择对端识别的sig算法，从而能够影响本端证书的选择
OPENSSL_EXPORT int SSL_CTX_check_private_key(const SSL_CTX *ctx); //检测私钥证书是否匹配
```



## 4 复用相关数据结构

复用最基础的数据结构就三个，这三个就是我们主要的研究对象。

### 4.1 SESSION

`SESSION`基本的数据结构，但是是关联SESSION ID还是SESSION CACHE是需要看SESSION里面的数据结构。实际上这意味着我们可以自由自在地往里面存储数据。

```c++
OPENSSL_EXPORT SSL_SESSION *SSL_SESSION_new(const SSL_CTX *ctx);
OPENSSL_EXPORT int SSL_SESSION_up_ref(SSL_SESSION *session);
OPENSSL_EXPORT void SSL_SESSION_free(SSL_SESSION *session); //减少引用计数，引用计数为0就触发free
OPENSSL_EXPORT uint64_t SSL_SESSION_get_time(const SSL_SESSION *session);//获取SESSION的颁发时间
OPENSSL_EXPORT size_t SSL_SESSION_get_master_key(const SSL_SESSION *session,
                                                 uint8_t *out, size_t max_out);//获取具体的衍生秘钥
OPENSSL_EXPORT int SSL_SESSION_should_be_single_use(const SSL_SESSION *session);//这个就很牛逼了，只能使用一次的SESSION
```



### 4.2 SESSION CACHE

和`SESSION`有什么区别？这个问题自然而然出现了，看注释应该是一样的，都是`SSL_session`结构。boringg ssl自己内置了一种SESSION CACHE的代码实现，使用的数据结构是内置的hashTable，原文引用于下

> For a server, the library implements a built-in internal session cache as an in-memory hash table. Servers may also use `SSL_CTX_sess_set_get_cb` and `SSL_CTX_sess_set_new_cb` to implement a custom external session cache. In particular, this may be used to share a session cache between multiple servers in a large deployment. An external cache may be used in addition to or instead of the internal one. Use `SSL_CTX_set_session_cache_mode` to toggle the internal cache.

```c++
#define SSL_SESS_CACHE_OFF 0x0000  //用来关停SESSION CACHE功能
struct ssl_session_st {
  explicit ssl_session_st(const bssl::SSL_X509_METHOD *method);
  ssl_session_st(const ssl_session_st &) = delete;
  ssl_session_st &operator=(const ssl_session_st &) = delete;

  CRYPTO_refcount_t references = 1;

  // ssl_version is the (D)TLS version that established the session.
  uint16_t ssl_version = 0;

  // group_id is the ID of the ECDH group used to establish this session or zero
  // if not applicable or unknown.
  uint16_t group_id = 0;

  // peer_signature_algorithm is the signature algorithm used to authenticate
  // the peer, or zero if not applicable or unknown.
  uint16_t peer_signature_algorithm = 0;

  // secret, in TLS 1.2 and below, is the master secret associated with the
  // session. In TLS 1.3 and up, it is the resumption PSK for sessions handed to
  // the caller, but it stores the resumption secret when stored on |SSL|
  // objects.
  int secret_length = 0;
  uint8_t secret[SSL_MAX_MASTER_KEY_LENGTH] = {0};

  // session_id - valid?
  unsigned session_id_length = 0;
  uint8_t session_id[SSL_MAX_SSL_SESSION_ID_LENGTH] = {0};
  // this is used to determine whether the session is being reused in
  // the appropriate context. It is up to the application to set this,
  // via SSL_new
  uint8_t sid_ctx_length = 0;
  uint8_t sid_ctx[SSL_MAX_SID_CTX_LENGTH] = {0};

  bssl::UniquePtr<char> psk_identity;

  // certs contains the certificate chain from the peer, starting with the leaf
  // certificate.
  bssl::UniquePtr<STACK_OF(CRYPTO_BUFFER)> certs;

  const bssl::SSL_X509_METHOD *x509_method = nullptr;

  // x509_peer is the peer's certificate.
  X509 *x509_peer = nullptr;

  // x509_chain is the certificate chain sent by the peer. NOTE: for historical
  // reasons, when a client (so the peer is a server), the chain includes
  // |peer|, but when a server it does not.
  STACK_OF(X509) *x509_chain = nullptr;

  // x509_chain_without_leaf is a lazily constructed copy of |x509_chain| that
  // omits the leaf certificate. This exists because OpenSSL, historically,
  // didn't include the leaf certificate in the chain for a server, but did for
  // a client. The |x509_chain| always includes it and, if an API call requires
  // a chain without, it is stored here.
  STACK_OF(X509) *x509_chain_without_leaf = nullptr;

  // verify_result is the result of certificate verification in the case of
  // non-fatal certificate errors.
  long verify_result = X509_V_ERR_INVALID_CALL;

  // timeout is the lifetime of the session in seconds, measured from |time|.
  // This is renewable up to |auth_timeout|.
  uint32_t timeout = SSL_DEFAULT_SESSION_TIMEOUT;

  // auth_timeout is the non-renewable lifetime of the session in seconds,
  // measured from |time|.
  uint32_t auth_timeout = SSL_DEFAULT_SESSION_TIMEOUT;

  // time is the time the session was issued, measured in seconds from the UNIX
  // epoch.
  uint64_t time = 0;

  const SSL_CIPHER *cipher = nullptr;

  CRYPTO_EX_DATA ex_data;  // application specific data

  // These are used to make removal of session-ids more efficient and to
  // implement a maximum cache size.
  SSL_SESSION *prev = nullptr, *next = nullptr;

  bssl::Array<uint8_t> ticket;

  bssl::UniquePtr<CRYPTO_BUFFER> signed_cert_timestamp_list;

  // The OCSP response that came with the session.
  bssl::UniquePtr<CRYPTO_BUFFER> ocsp_response;

  // peer_sha256 contains the SHA-256 hash of the peer's certificate if
  // |peer_sha256_valid| is true.
  uint8_t peer_sha256[SHA256_DIGEST_LENGTH] = {0};

  // original_handshake_hash contains the handshake hash (either SHA-1+MD5 or
  // SHA-2, depending on TLS version) for the original, full handshake that
  // created a session. This is used by Channel IDs during resumption.
  uint8_t original_handshake_hash[EVP_MAX_MD_SIZE] = {0};
  uint8_t original_handshake_hash_len = 0;

  uint32_t ticket_lifetime_hint = 0;  // Session lifetime hint in seconds

  uint32_t ticket_age_add = 0;

  // ticket_max_early_data is the maximum amount of data allowed to be sent as
  // early data. If zero, 0-RTT is disallowed.
  uint32_t ticket_max_early_data = 0;

  // early_alpn is the ALPN protocol from the initial handshake. This is only
  // stored for TLS 1.3 and above in order to enforce ALPN matching for 0-RTT
  // resumptions. For the current connection's ALPN protocol, see
  // |alpn_selected| on |SSL3_STATE|.
  bssl::Array<uint8_t> early_alpn;

  // local_application_settings, if |has_application_settings| is true, is the
  // local ALPS value for this connection.
  bssl::Array<uint8_t> local_application_settings;

  // peer_application_settings, if |has_application_settings| is true, is the
  // peer ALPS value for this connection.
  bssl::Array<uint8_t> peer_application_settings;

  // extended_master_secret is whether the master secret in this session was
  // generated using EMS and thus isn't vulnerable to the Triple Handshake
  // attack.
  bool extended_master_secret : 1;

  // peer_sha256_valid is whether |peer_sha256| is valid.
  bool peer_sha256_valid : 1;  // Non-zero if peer_sha256 is valid

  // not_resumable is used to indicate that session resumption is disallowed.
  bool not_resumable : 1;

  // ticket_age_add_valid is whether |ticket_age_add| is valid.
  bool ticket_age_add_valid : 1;

  // is_server is whether this session was created by a server.
  bool is_server : 1;

  // is_quic indicates whether this session was created using QUIC.
  bool is_quic : 1;

  // has_application_settings indicates whether ALPS was negotiated in this
  // session.
  bool has_application_settings : 1;

  // quic_early_data_context is used to determine whether early data must be
  // rejected when performing a QUIC handshake.
  bssl::Array<uint8_t> quic_early_data_context;

 private:
  ~ssl_session_st();
  friend void SSL_SESSION_free(SSL_SESSION *);
};
```



数据结构看完了，看看`SESSION CACHE`的管理是怎么做的，几个关键函数如下，其中：

+ 从`session_id`拓展到`hash`实际上使用了`session_id`的前四字节

  ```c++
  uint32_t ssl_hash_session_id(Span<const uint8_t> session_id) {
    // Take the first four bytes of |session_id|. Session IDs are generated by the
    // server randomly, so we can assume even using the first four bytes results
    // in a good distribution.
    uint8_t tmp_storage[sizeof(uint32_t)];
    if (session_id.size() < sizeof(tmp_storage)) {
      OPENSSL_memset(tmp_storage, 0, sizeof(tmp_storage));
      OPENSSL_memcpy(tmp_storage, session_id.data(), session_id.size());
      session_id = tmp_storage;
    }
  
    uint32_t hash =
        ((uint32_t)session_id[0]) |
        ((uint32_t)session_id[1] << 8) |
        ((uint32_t)session_id[2] << 16) |
        ((uint32_t)session_id[3] << 24);
  
    return hash;
  }
  ```

  

+ 有了`hash+session_id+session_ctx->sessions`最后实际上是个比较函数，BORING SSL最后使用的比较函数是个宏，展开来这个宏如下。所以我们得看看LHASH到底是个什么数据结构，因为`ssl->session_ctx->sessions`的数据结构是`LHASH_OF(SSL_SESSION) *sessions = nullptr;`。这个东西在OPENSSL里面也有，说白了就是hash表，但是一个大锁对应一个HASH表可太狠了，所以我们得看看`ssl_lookup_session`的时候发生了什么

  ```c++
  // ssl_lookup_session looks up |session_id| in the session cache and sets
  // |*out_session| to an |SSL_SESSION| object if found.
  static enum ssl_hs_wait_t ssl_lookup_session(
      SSL_HANDSHAKE *hs, UniquePtr<SSL_SESSION> *out_session,
      Span<const uint8_t> session_id) {
    SSL *const ssl = hs->ssl;
    out_session->reset();
  
    if (session_id.empty() || session_id.size() > SSL_MAX_SSL_SESSION_ID_LENGTH) {
      return ssl_hs_ok;//session id不合法，不是合理的session id就直接放弃
    }
  
    UniquePtr<SSL_SESSION> session;
    // Try the internal cache, if it exists.
    if (!(ssl->session_ctx->session_cache_mode &
          SSL_SESS_CACHE_NO_INTERNAL_LOOKUP)) {
      uint32_t hash = ssl_hash_session_id(session_id);//先通过session_id计算出来hash
      auto cmp = [](const void *key, const SSL_SESSION *sess) -> int {
        Span<const uint8_t> key_id =
            *reinterpret_cast<const Span<const uint8_t> *>(key);
        Span<const uint8_t> sess_id =
            MakeConstSpan(sess->session_id, sess->session_id_length);
        return key_id == sess_id ? 0 : 1;
      };
      MutexReadLock lock(&ssl->session_ctx->lock);  //读锁开始锁SSL连接相关的session-ctx的锁了，然后升高ssl->session-ctx->session的reference count。但是这个锁太大了！去找一下这个锁的初始化的位置，这个锁太狠了
      // |lh_SSL_SESSION_retrieve_key| returns a non-owning pointer.
      session = UpRef(lh_SSL_SESSION_retrieve_key(ssl->session_ctx->sessions,
                                                  &session_id, hash, cmp));
      // TODO(davidben): This should probably move it to the front of the list.
    }
  
    // Fall back to the external cache, if it exists.
    if (!session && ssl->session_ctx->get_session_cb != nullptr) {
      int copy = 1;
      session.reset(ssl->session_ctx->get_session_cb(ssl, session_id.data(),
                                                     session_id.size(), &copy));
      if (!session) {
        return ssl_hs_ok;
      }
  
      if (session.get() == SSL_magic_pending_session_ptr()) {
        session.release();  // This pointer is not actually owned.
        return ssl_hs_pending_session;
      }
  
      // Increment reference count now if the session callback asks us to do so
      // (note that if the session structures returned by the callback are shared
      // between threads, it must handle the reference count itself [i.e. copy ==
      // 0], or things won't be thread-safe).
      if (copy) {
        SSL_SESSION_up_ref(session.get());
      }
  
      // Add the externally cached session to the internal cache if necessary.
      if (!(ssl->session_ctx->session_cache_mode &
            SSL_SESS_CACHE_NO_INTERNAL_STORE)) {
        SSL_CTX_add_session(ssl->session_ctx.get(), session.get());
      }
    }
  
    if (session && !ssl_session_is_time_valid(ssl, session.get())) {
      // The session was from the cache, so remove it.
      SSL_CTX_remove_session(ssl->session_ctx.get(), session.get());
      session.reset();
    }
  
    *out_session = std::move(session);
    return ssl_hs_ok;
  }
  ```

+ 上面看完了`LOOKUP`我们看看new SESSION的时候，也就是添加SESSION的时候发生了什么？一个ssl的session最多存储多少个？

  ```c++
  int SSL_CTX_add_session(SSL_CTX *ctx, SSL_SESSION *session) {
    // Although |session| is inserted into two structures (a doubly-linked list
    // and the hash table), |ctx| only takes one reference.
    UniquePtr<SSL_SESSION> owned_session = UpRef(session);
  
    SSL_SESSION *old_session;
    MutexWriteLock lock(&ctx->lock);//还是一把大写锁啊，蛋疼
    if (!lh_SSL_SESSION_insert(ctx->sessions, &old_session, session)) {
      return 0;
    }
    // |ctx->sessions| took ownership of |session| and gave us back a reference to
    // |old_session|. (|old_session| may be the same as |session|, in which case
    // we traded identical references with |ctx->sessions|.)
    owned_session.release();
    owned_session.reset(old_session);
  
    if (old_session != NULL) {
      if (old_session == session) {
        // |session| was already in the cache. There are no linked list pointers
        // to update.
        return 0;
      }
  
      // There was a session ID collision. |old_session| was replaced with
      // |session| in the hash table, so |old_session| must be removed from the
      // linked list to match.
      SSL_SESSION_list_remove(ctx, old_session);
    }
  
    SSL_SESSION_list_add(ctx, session);
  
    // Enforce any cache size limits.
    if (SSL_CTX_sess_get_cache_size(ctx) > 0) {
      while (lh_SSL_SESSION_num_items(ctx->sessions) >
             SSL_CTX_sess_get_cache_size(ctx)) {
        if (!remove_session_lock(ctx, ctx->session_cache_tail, 0)) {
          break;
        }
      }
    }
  
    return 1;
  }
  ```

  

+ 上面看完了`NEW`我们看看删除SESSION的时候发生了什么

  ```c++
  static int remove_session_lock(SSL_CTX *ctx, SSL_SESSION *session, int lock) {
    int ret = 0;
  
    if (session != NULL && session->session_id_length != 0) {
      if (lock) {
        CRYPTO_MUTEX_lock_write(&ctx->lock);       //先加锁，先把锁给锁上避免争用
      }
      SSL_SESSION *found_session = lh_SSL_SESSION_retrieve(ctx->sessions, session);  //好了，获取对应的SESSION看是不是还存在，不存在就说明已经被其他的线程给删除了，那我们就没啥事情了，但是需要释放锁。
      if (found_session == session) {
        ret = 1;
        found_session = lh_SSL_SESSION_delete(ctx->sessions, session);//从OPENSSL的hash表里面删除掉这个session
        SSL_SESSION_list_remove(ctx, session);//hash表删除掉可没玩，还需要从链表上把session删除掉
      }
  
      if (lock) {
        CRYPTO_MUTEX_unlock_write(&ctx->lock);
      }
  
      if (ret) {  //ret为1，说明是当前线程执行的删除操作，因此需要释放内存
        if (ctx->remove_session_cb != NULL) {
          ctx->remove_session_cb(ctx, found_session);
        }
        SSL_SESSION_free(found_session);//释放掉内存
      }
    }
  
    return ret;
  }
  
  static void SSL_SESSION_list_remove(SSL_CTX *ctx, SSL_SESSION *session) {  //一个基础的链表，直接删除就成了
    if (session->next == NULL || session->prev == NULL) {
      return;
    }
  
    if (session->next == (SSL_SESSION *)&ctx->session_cache_tail) {
      // last element in list
      if (session->prev == (SSL_SESSION *)&ctx->session_cache_head) {
        // only one element in list
        ctx->session_cache_head = NULL;
        ctx->session_cache_tail = NULL;
      } else {
        ctx->session_cache_tail = session->prev;
        session->prev->next = (SSL_SESSION *)&(ctx->session_cache_tail);
      }
    } else {
      if (session->prev == (SSL_SESSION *)&ctx->session_cache_head) {
        // first element in list
        ctx->session_cache_head = session->next;
        session->next->prev = (SSL_SESSION *)&(ctx->session_cache_head);
      } else {  // middle of list
        session->next->prev = session->prev;
        session->prev->next = session->next;
      }
    }
    session->prev = session->next = NULL;//看看这一步操作，还是把当前要去掉的session的前和后置空了，对一部分人，教科书级别的打脸啊。
  }
  ```

  

所以可以总结`session_ctx->lock`的管辖范围，一个大锁管了一堆，虽然有lazy delete但是，没蛋用。能够看出来，实际上BORING SSL和OPENSSL都一样，都是一把大锁加个链表存储

### 4.3 SESSION TICKET

和SESSION CACHE不同，RFC5077里面给出了SESSION TICKET的实现。对于BORING SSL而言，SESSION TICKET也是支持的。

> On the server, tickets are encrypted and authenticated with a secret key. By default, an `SSL_CTX` will manage session ticket encryption keys by generating them internally and rotating every 48 hours. Tickets are minted and processed transparently. The following functions may be used to configure a persistent key or implement more custom behavior, including key rotation and sharing keys between multiple servers in a large deployment. There are three levels of customisation possible:
>
> 1) One can simply set the keys with `SSL_CTX_set_tlsext_ticket_keys`. 2) One can configure an `EVP_CIPHER_CTX` and `HMAC_CTX` directly for encryption and authentication. 3) One can configure an `SSL_TICKET_AEAD_METHOD` to have more control and the option of asynchronous decryption.

相比较而言，我更喜欢`session ticket`，因为这东西完全不在本地存储软件，省大了去的事啦！而且只要逻辑确定了，避开锁争用的消耗就太棒啦！看一下具体的创建session_tickets的函数

```c++
static bool add_new_session_tickets(SSL_HANDSHAKE *hs, bool *out_sent_tickets) {
  SSL *const ssl = hs->ssl;
  if (// If the client doesn't accept resumption with PSK_DHE_KE, don't send a
      // session ticket.
      !hs->accept_psk_mode ||
      // We only implement stateless resumption in TLS 1.3, so skip sending
      // tickets if disabled.
      (SSL_get_options(ssl) & SSL_OP_NO_TICKET)) {  //判断能不能颁发session，看来boring ssl在老版本根本没做这工作
    *out_sent_tickets = false;
    return true;
  }

  // TLS 1.3 recommends single-use tickets, so issue multiple tickets in case
  // the client makes several connections before getting a renewal.
  static const int kNumTickets = 2;    //每次颁发两个single-use ticket，来保证能够多次复用，不过我很好奇这个single-use怎么做的

  // Rebase the session timestamp so that it is measured from ticket
  // issuance.
  ssl_session_rebase_time(ssl, hs->new_session.get());   //校定时间

  for (int i = 0; i < kNumTickets; i++) {
    UniquePtr<SSL_SESSION> session(
        SSL_SESSION_dup(hs->new_session.get(), SSL_SESSION_INCLUDE_NONAUTH));
    if (!session) {
      return false;
    }

    if (!RAND_bytes((uint8_t *)&session->ticket_age_add, 4)) {  //随机生成ticket_age_add
      return false;
    }
    session->ticket_age_add_valid = true;
    bool enable_early_data =
        ssl->enable_early_data &&
        (!ssl->quic_method || !ssl->config->quic_early_data_context.empty());
    if (enable_early_data) {  //校定是不是需要EARLYDATA拓展
      // QUIC does not use the max_early_data_size parameter and always sets it
      // to a fixed value. See draft-ietf-quic-tls-22, section 4.5.
      session->ticket_max_early_data =
          ssl->quic_method != nullptr ? 0xffffffff : kMaxEarlyDataAccepted;
    }

    static_assert(kNumTickets < 256, "Too many tickets");
    uint8_t nonce[] = {static_cast<uint8_t>(i)};

    ScopedCBB cbb;
    CBB body, nonce_cbb, ticket, extensions;
    if (!ssl->method->init_message(ssl, cbb.get(), &body,
                                   SSL3_MT_NEW_SESSION_TICKET) ||
        !CBB_add_u32(&body, session->timeout) ||
        !CBB_add_u32(&body, session->ticket_age_add) ||
        !CBB_add_u8_length_prefixed(&body, &nonce_cbb) ||
        !CBB_add_bytes(&nonce_cbb, nonce, sizeof(nonce)) ||
        !CBB_add_u16_length_prefixed(&body, &ticket) ||
        !tls13_derive_session_psk(session.get(), nonce) ||
        !ssl_encrypt_ticket(hs, &ticket, session.get()) ||
        !CBB_add_u16_length_prefixed(&body, &extensions)) {
      return false;
    }

    if (enable_early_data) {  //根据是否需要加入early_data来确定是不是加拓展
      CBB early_data;
      if (!CBB_add_u16(&extensions, TLSEXT_TYPE_early_data) ||
          !CBB_add_u16_length_prefixed(&extensions, &early_data) ||
          !CBB_add_u32(&early_data, session->ticket_max_early_data) ||
          !CBB_flush(&extensions)) {
        return false;
      }
    }

    // Add a fake extension. See draft-davidben-tls-grease-01.
    if (!CBB_add_u16(&extensions,
                     ssl_get_grease_value(hs, ssl_grease_ticket_extension)) ||
        !CBB_add_u16(&extensions, 0 /* empty */)) {
      return false;
    }

    if (!ssl_add_message_cbb(ssl, cbb.get())) {
      return false;
    }
  }

  *out_sent_tickets = true;
  return true;
}


```



最后我们提一句single-use这个事情，TLS1.3 RFC对于SINGLE-USE是维持一个SESSION TICKET数据库，如果用到了就直接删除，保证不会再出现复用。那么BORING SSL怎么做的呢？我们就看看BORING SSL处理SESSION TICKET的逻辑，换言之，只要是TLS1.3之后，包括TLS1.3的版本，必然是SINGLE-USE的，这个假设可以说是相当错误，实际上没必须如此的设定。

```c++
int SSL_SESSION_should_be_single_use(const SSL_SESSION *session) {
  return ssl_session_protocol_version(session) >= TLS1_3_VERSION;
}
```

那么如何实现SINGLE-USE的呢？遗憾的是，这个东西BORING SSL和OPENSSL SSL都没实现：

> Note also, in TLS 1.2 and earlier, offering sessions allows passive observers to correlate different client connections. TLS 1.3 and later fix this, provided clients use sessions at most once. Session caches are managed by the caller in BoringSSL, so this must be implemented externally. See `SSL_SESSION_should_be_single_use` for details.









## 5 拓展相关数据结构

### 5.1 ALPN

### 5.2 NPN



## 结尾

唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)