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

## 底层实现

Envoy 当前使用 BoringSSL 作为其 TLS 提供者。

## FIPS 140-2

使用 Bazel 选项 `--define boringssl=fips` 进行构建, 遵循 [BoringCrypto 模块安全策略](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3678.pdf)的构建指南，BoringSSL 可以配置为 [FIPS-complianthttps://boringssl.googlesource.com/boringssl/+/master/crypto/fipsmodule/FIPS.md) 模式。目前，此选项仅在 Linux-x86_64 上可用。

要验证 FIPS 构建是否正确，可以检查 `--version` 输出中是否含有 `BoringSSL-FIPS`。

请注意，仅使用 FIPS-compliant 模块是达成 FIPS 合规的必要条件，但不是充分条件，
根据不同情况，可能还需要采取其他措施。这些额外的措施可能包括仅使用经批准的算法，
或者仅使用由以 FIPS-approved 模式运行的模块生成的私钥。更多信息，请参阅
 [BoringCrypto 模块安全策略](https://csrc.nist.gov/CSRC/media/projects/cryptographic-module-validation-program/documents/security-policies/140sp3678.pdf)或 [认证的 CMVP 实验室](https://csrc.nist.gov/projects/testing-laboratories)。

另外，请知悉，FIPS-compliant 构建所依赖的 BoringSSL 版本早于非 FIPS 构建的版本，
并且不支持最新的 QUIC API。

# JSON Web Token（JWT）认证机制

- [HTTP过滤器的配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/jwt_authn_filter#config-http-filters-jwt-authn)说明。

JWT认证过滤器用于检查传入请求中的[JSON Web Token（JWT）](https://tools.ietf.org/html/rfc7519)
是否有效。该过滤器通过验证JWT的签名、接收者以及发行者来确
保JWT的有效性，这些验证基于[HTTP过滤器的配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/jwt_authn_filter#config-http-filters-jwt-authn)。JWT认证过
滤器可以配置为：一旦发现JWT无效，立即拒绝请求；或者将JWT
的有效载荷传递给后续过滤器，延后做出处理决定。

JWT认证过滤器能够根据请求的不同条件来检查JWT。例如，它
可以被设置为只在特定路径上进行JWT检查，从而允许某些路径不
进行JWT认证。这特别适用于无需JWT认证即可公开访问的路径。

JWT认证过滤器能够从请求的不同位置提取JWT，并且能够对同一请求施加
多重JWT校验条件。用于JWT签名验证的[JSON Web Key Set（JWKS）](https://tools.ietf.org/html/rfc7517)
可以有两种配置方式：直接内嵌于过滤器配置，或者通过HTTP/HTTPS
从远程服务器获取。

此外，JWT认证过滤器还能将成功验证的JWT的头部和有效载荷记录到
[Dynamic State](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/advanced/data_sharing_between_filters#arch-overview-data-sharing-between-filters)
中。这样，后续过滤器就能利用这些信息，基于JWT有效载荷做出决策。

# 外部授权

- [网络过滤器配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/ext_authz_filter#config-network-filters-ext-authz)

- [HTTP过滤器配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/ext_authz_filter#config-http-filters-ext-authz)

外部授权过滤器负责调用授权服务，以确定传入请求是否获得授权。该过
滤器既可以作为网络过滤器配置，也可以作为HTTP过滤器配置，还可以两
者同时使用。如果网络过滤器判定请求未获授权，那么将关闭连接。相应
地，如果HTTP过滤器判定请求未授权，则会以403 Forbidden（禁止）
状态码拒绝请求。

> **提示**
>
> 建议将这些过滤器设置为过滤器链的首个环节，这样可以确保在其他过滤器处理
> 请求前先进行授权。

外部授权服务的集群配置可以是静态的，也可以通过[Cluster Discovery Service](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cds#config-cluster-manager-cds)
动态设置。如果外部服务在处理请求时不可用，那么是否授权该请求将由
[网络过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/ext_authz/v3/ext_authz.proto#envoy-v3-api-msg-extensions-filters-network-ext-authz-v3-extauthz)
或
[HTTP过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/http/ext_authz/v3/ext_authz.proto#envoy-v3-api-msg-extensions-filters-http-ext-authz-v3-extauthz)
中的failure_mode_allow配置决定。如果此项配置为true，
则请求在失败时也会被放行（fail-open）；如果为false，则请求会被拒绝。
默认情况下，此配置为false。

## 服务定义

以下服务定义用于将流量上下文传送到外部授权服务。授权服务所接收的请
求内容由[CheckRequest](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/external_auth.proto#envoy-v3-api-msg-service-auth-v3-checkrequest)定义。


- [Attribute context (proto)](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/attribute_context.proto)
    - [service.auth.v3.AttributeContext](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/attribute_context.proto#service-auth-v3-attributecontext)
    - [service.auth.v3.AttributeContext.Peer](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/attribute_context.proto#service-auth-v3-attributecontext-peer)
    - [service.auth.v3.AttributeContext.Request](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/attribute_context.proto#service-auth-v3-attributecontext-request)
    - [service.auth.v3.AttributeContext.HttpRequest](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/attribute_context.proto#service-auth-v3-attributecontext-httprequest)
    - [service.auth.v3.AttributeContext.TLSSession](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/attribute_context.proto#service-auth-v3-attributecontext-tlssession)
- [Authorization service (proto)](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/external_auth.proto)
    - [service.auth.v3.CheckRequest](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/external_auth.proto#service-auth-v3-checkrequest)
    - [service.auth.v3.DeniedHttpResponse](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/external_auth.proto#service-auth-v3-deniedhttpresponse)
    - [service.auth.v3.OkHttpResponse](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/external_auth.proto#service-auth-v3-okhttpresponse)
    - [service.auth.v3.CheckResponse](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/service/auth/v3/external_auth.proto#service-auth-v3-checkresponse)
