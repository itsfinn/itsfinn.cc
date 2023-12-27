---
title: "Envoy 介绍之(五)"
date: 2023-12-27T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---

原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(安全)
<!--more-->

# TLS

Envoy 不仅支持监听器中的 TLS 终止，还能在与上游集群连接时发起 TLS。该支持使 Envoy 能够充当满足现代
网络服务要求的标准边缘代理，并能与要求高级 TLS 设置（如 TLS1.2、SNI 等）的外部服务建立连接。Envoy 提供
以下 TLS 功能：
  
- 可定制的加密套件：每个 TLS 监听器和客户端都可以自定义支持的加密套件。
- 客户端证书：上游或客户端连接可以提供客户端证书，同时进行服务器证书验证(原文：Client certificates: Upstream/client connections can present a client certificate in addition to server certificate verification.)。
- 证书验证与锁定：证书验证的方式包括链路验证、主体名称验证和哈希锁定。
- 证书撤销：Envoy 能够根据[提供](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/common.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-certificatevalidationcontext-crl)的证书撤销列表（CRL）检查对等方证书。
- ALPN：TLS 监听器支持应用层协议协商（ALPN）。HTTP 连接管理器利用此信息（结合协议推断）确定客户端是
  否使用 HTTP/1.1 或 HTTP/2。
- SNI：SNI 技术在服务器（监听器）和客户端（上游）连接中都得到支持。
- 会话恢复：服务器连接可以通过 TLS 会话票据（参照 [RFC 5077](https://www.ietf.org/rfc/rfc5077.txt)）
  来恢复先前的会话。此类会话恢复可以在热重启中进行，
  也可以在并行运行的 Envoy 实例间进行，这在前端代理配置中尤为有用。
- BoringSSL 私钥方法：TLS 私钥操作（包括签名和解密）可通过[扩展](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/common.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-privatekeyprovider)异步执行，使得 Envoy 能够支持多种密钥
  管理方案（例如 TPM）和 TLS 加速。该机制采用 [BoringSSL 的私钥方法接口](https://github.com/google/boringssl/blob/c0b4c72b6d4c6f4828a373ec454bd646390017d4/include/openssl/ssl.h#L1169)。
- OCSP Stapling：可以将在线证书状态协议（OCSP）的响应捆绑到证书上。
