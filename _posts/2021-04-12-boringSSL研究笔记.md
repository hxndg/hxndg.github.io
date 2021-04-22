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

#### 1.2.1 TLS连接结构

可以看出来SSL_HANDSHAKE结构和具体的握手状态是相关的，具体的握手状态在自动机里面才会用到，先不过于关注这几个东西。BoringSSL添加了一个SSL_CONFIG结构用于完成握手之后给后面连接保存必要的信息。 `SSL`并不是线程安全的，任何时刻只能有一个线程使用该数据结构。具体的配置可以直接对`SSL_CTX`结构或者世界对`SSL`结构配置。

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








## 结尾
唉，尴尬

![狗头的赞赏码.jpg](https://i.loli.net/2020/08/27/BFHNyfpx3EsIDUG.jpg)
