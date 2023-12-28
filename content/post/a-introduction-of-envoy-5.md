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

Envoy 官网介绍文档的中文翻译(安全):TLS、JWT、外部授权、基于角色访问控制、威胁模型、扩展安全、外部依赖、漏洞奖励计划
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

## 启用证书验证

除非在验证上下文中明确指定了一个或多个受信任的授权证书，否则不会对上游和下游的连接进行证书验证。

### 配置示例

```yaml
static_resources:
  listeners:
  - name: listener_0
    address: {socket_address: {address: 127.0.0.1, port_value: 10000}}
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            virtual_hosts:
            - name: default
              domains: ["*"]
              routes:
              - match: {prefix: "/"}
                route:
                  cluster: some_service
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_certificates:
            - certificate_chain: {filename: "certs/servercert.pem"}
              private_key: {filename: "certs/serverkey.pem"}
            validation_context:
              trusted_ca:
                filename: certs/cacert.pem
  clusters:
  - name: some_service
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: some_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 1234
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        common_tls_context:
          tls_certificates:
          - certificate_chain: {"filename": "certs/servercert.pem"}
            private_key: {"filename": "certs/serverkey.pem"}
            ocsp_staple: {"filename": "certs/server_ocsp_resp.der"}
          validation_context:
            match_typed_subject_alt_names:
            - san_type: DNS
              matcher:
                exact: "foo"
            trusted_ca:
              filename: /etc/ssl/certs/ca-certificates.crt
```

在 Debian 系统中，*/etc/ssl/certs/ca-certificates.crt* 是系统 CA 包的默认存放路径。
通过配置 
[trusted_ca](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/common.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-certificatevalidationcontext-trusted-ca) 
和 [match_typed_subject_alt_names](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/common.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-certificatevalidationcontext-match-typed-subject-alt-names)，
Envoy 能够按照 Debian 系统中 cURL 的标准方式，
验证 127.0.0.1:1234 的服务器身份是否为“foo”。在 Linux 和 BSD 系统中，系统 CA 包的常见路径有：

- /etc/ssl/certs/ca-certificates.crt (Debian/Ubuntu/Gentoo 等)
- /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem (CentOS/RHEL 7)
- /etc/pki/tls/certs/ca-bundle.crt (Fedora/RHEL 6)
- /etc/ssl/ca-bundle.pem (OpenSUSE)
- /usr/local/etc/ssl/cert.pem (FreeBSD)
- /etc/ssl/cert.pem (OpenBSD)

更多关于 TLS 的配置选项，可参阅 
[UpstreamTlsContexts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-upstreamtlscontext) 
和 [DownstreamTlsContexts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-downstreamtlscontext)
的官方文档。

> **注意事项**
>
> 仅当指定了 [trusted_ca](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/common.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-certificatevalidationcontext-trusted-ca) 时，Envoy 会对提出的证书的证书链进行验证，但不对其主体名称、哈希等进行验证。
> 具体部署的情况通常还需要其他验证上下文配置。

### 自定义证书验证器

上述配置适用于 Envoy 的“默认”证书验证器。
Envoy 还支持自定义验证器，这些验证器可以在 `envoy.tls.cert_validator` 扩展类别中配置，
并且可以在 [CertificateValidationContext](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/common.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-certificatevalidationcontext) 中设置。

例如，Envoy 可以根据 [SPIFFE](https://github.com/spiffe/spiffe) 规范来验证对等方的证书，并且可以在同一个监听器或集群中支持多个信任捆绑包。
详细信息请参考 [custom_validator_config](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/common.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-certificatevalidationcontext-custom-validator-config) 字段的说明文档。

## 证书选择

[DownstreamTlsContexts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-downstreamtlscontext)
支持配置多个 TLS 证书，这些证书可能是为多个服务器名称模式混合使用的 RSA 和 P-256 ECDSA 证书。

证书配置与加载规则：

- 在握手过程中，DNS SANs 或主体通用名称将被用作服务器名称模式以匹配 SNI。如果证书中包含 DNS SANs，主体通用名称将不被使用。
- 全域名（FQDN）如 “test.example.com” 和通配符如 “*.example.com” 可以同时被视为有效，并且这将被加载为两个不同的服务器名称模式。
- 若为同一名称或名称模式指定了多个同类型（RSA 或 ECDSA）的证书，系统将使用第一个被加载的证书。
- 不接受非 P-256 的服务器 ECDSA 证书。
- 不能在同一个 [DownstreamTlsContexts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-downstreamtlscontext) 中混合使用静态和 SDS 证书。

证书选择规则：

- 当客户端支持服务器名称指示(SNI)，如SNI为“test.example.com”时，系统将寻找与之完全一致
  的证书。若该证书遵循在线证书状态协议(OCSP)策略且密钥类型匹配，则选用此证书进行安全握手。
  如果证书符合OCSP策略，但是密钥类型为RSA，而客户端支持ECDSA，此证书将被列为备选，系统
  会持续搜索直至找到完全匹配的证书或证书被搜寻完毕。若无完美匹配的证书，将使用备选证书
  进行安全握手。
- 若客户端支持SNI，但未找到与SNI完全匹配的证书，系统将尝试匹配含有通配符的服务器名称。
  例如，SNI为“test.example.com”时，“test.example.com”的证书将优先于“*.example.com”证书。
  通配符只能匹配域名的一级，因此“*.com”不能匹配“test.example.com”。接下来，系统会对每个
  证书执行OCSP策略和密钥类型检查，与精确匹配SNI后的流程相同。
- 如果没有从匹配通配符的证书中选出证书，且存在备选证书，则选用备选证书进行安全握手。
  如果没有备选证书，系统将检查是否启用了
  [在SNI不匹配时全面扫描证书](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-downstreamtlscontext-full-scan-certs-on-sni-mismatch)
  选项；如果启用，则扫描全部证书，否则默认选用列表中的第一个证书进行安全握手。
- 若客户端未提供SNI信息，无论
  [在SNI不匹配时全面扫描证书](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-downstreamtlscontext-full-scan-certs-on-sni-mismatch)
  是否为真，都将执行全面扫描。
- 全面扫描将对每个证书进行OCSP策略和密钥类型检查，这与精确SNI匹配的流程相同。
  如果没有合适的证书，则默认选用列表中的第一个证书。
- 目前系统仅支持两种密钥算法：RSA和P-256 ECDSA。如果客户端支持P-256 ECDSA算法，系统
  将优先选用P-256 ECDSA证书而不是RSA证书。默认选用的证书可能导致安全握手失败，例如，
  当客户端仅支持RSA证书而服务器证书仅支持ECDSA算法时。
- 最终选用的证书必须遵循OCSP策略。如果无法找到符合此策略的证书，将拒绝建立连接。

> **注意**
>
> 通过支持基于服务器名称指示(SNI)的证书选择机制，我们可以为多个主机名配置大量证书。
> 系统引入了一个配置项
>  [在SNI不匹配时全面扫描证书](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-downstreamtlscontext-full-scan-certs-on-sni-mismatch),
> 用于决定当客户端提供SNI信息时，
> 如果发生SNI不匹配，是否继续执行完整的证书扫描。SNI不匹配的情况有两种：一是没有找到与SNI
> 相匹配的证书，二是找到了匹配的证书但它们不符合在线证书状态协议(OCSP)策略。该配置默认关闭，
> 即默认情况下不执行完整扫描。可以使用运行时标志`envoy.reloadable_features.no_full_scan_certs_on_sni_mismatch`
> 来覆盖这一默认设置。如果开启了完整扫描，在SNI不匹配的情况下，系统将检查整个证书列表，
> 这可能因为O(n)的复杂度而引起潜在的DoS攻击风险。

当前，[UpstreamTlsContexts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-upstreamtlscontext)
仅支持配置单个TLS证书。

## 密钥发现服务(SDS)

TLS证书可以直接在静态资源中指定，也可以通过远程服务获取。对于静态资源中的TLS证书，
可以通过[从文件系统源获取SDS配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/security/secret#xds-certificate-rotation)或接收来自SDS服务器的推送来实现轮换更新。详情请参见[SDS](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/security/secret#config-secret-discovery-service)文档。

## OCSP 嵌入(OCSP Stapling)

[DownstreamTlsContexts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-downstreamtlscontext)
支持在TLS握手过程中添加在线证书状态协议（OCSP）的响应到证书中。通过
"ocsp_staple"字段，管理员可以为每个证书配置预生成的OCSP响应。注意，一个OCSP响应不能用于多个证书。提供的OCSP响
应必须有效并证明证书尚未吊销。尽管过期的OCSP响应仍然可以接受，但根据具体的OCSP固定策略，这可能会引起下游连接的错误。

[DownstreamTlsContexts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-downstreamtlscontext)
还提供了"ocsp_staple_policy"字段，用于控制当相关OCSP响应丢失或过期时，Envoy是应该停止使用该证书还
是继续进行连接而不添加OCSP响应。对于那些标记为"必须添加OCSP响应"的证书，无论固定策略如何，都必须要有有效的OCSP响
应。实际上，带有"必须添加OCSP响应"标记的证书会使得Envoy按照OCSP固定策略必须为MUST_STAPLE来操作。一旦OCSP响应
过期，Envoy将不会用这类证书建立新的连接。

未表明支持OCSP Stapling的TLS请求，其OCSP响应将不会通过status_request扩展进行捆绑。

以下运行时参数用于调整OCSP响应的规定并可以覆盖默认的OCSP策略，其默认值为true。

- `envoy.reloadable_features.require_ocsp_response_for_must_staple_certs`:
关闭该参数将允许管理员在配置中对于必须绑定OCSP响应的证书忽略OCSP响应。

- `envoy.reloadable_features.check_ocsp_policy`: 关闭该参数将停止OCSP策略的检查。
  OCSP响应，哪怕已经过期，只要客户端支持，也会在可用的情况下进行捆绑。如果没有OCSP响应，则不进行捆绑。

对于[UpstreamTlsContexts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-upstreamtlscontext)，OCSP响应将不予考虑。

## 认证过滤器

Envoy支持一种网络过滤器，它能够通过获取REST VPN服务中的认证主体来实施TLS客户端认证。
此过滤器对比提交的客户端证书哈希与认证主体列表，以判定是否允许建立连接。
用户还可以选择配置IP白名单。该功能为Web基础架构提供了实现边缘代理VPN支持的可能。

客户端TLS认证的过滤器[配置参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/client_ssl_auth_filter#config-network-filters-client-ssl-auth)

## 自定义握手器扩展

[CommonTlsContext](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-field-extensions-transport-sockets-tls-v3-commontlscontext-custom-handshaker)提供了一个名为`custom_handshaker`扩展，允许用户彻底重写SSL握手的行为。
这非常适用于实现一些难以通过回调函数实现的TLS特定行为。使用私钥方法时，无需编写自定义握手器，
详见之前提到的[私钥方法接口](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/security/ssl#arch-overview-ssl)。

为避免重复实现[Ssl::ConnectionInfo](https://github.com/envoyproxy/envoy/blob/64bd6311bcc8f5b18ce44997ae22ff07ecccfe04/include/envoy/ssl/connection.h#L19)
接口的所有内容，自定义实现可以考虑基于
[Envoy::Extensions::TransportSockets::Tls::SslHandshakerImpl](https://github.com/envoyproxy/envoy/blob/64bd6311bcc8f5b18ce44997ae22ff07ecccfe04/source/extensions/transport_sockets/tls/ssl_handshaker.h#L40)
进行扩展。

自定义握手器必须通过[HandshakerCapabilities](https://github.com/envoyproxy/envoy/blob/64bd6311bcc8f5b18ce44997ae22ff07ecccfe04/include/envoy/ssl/handshaker.h#L68-L89)
清楚地声明它们负责管理哪些TLS特性。
Envoy默认的握手器会处理其他所有部分。

在[相关测试代码](https://github.com/envoyproxy/envoy/blob/64bd6311bcc8f5b18ce44997ae22ff07ecccfe04/test/extensions/transport_sockets/tls/handshaker_test.cc#L174-L184)中有一个实用的握手器示例，名为`SslHandshakerImplForTest`，它展示了如何处理
特殊的`SSL_ERROR`以及如何执行回调。

## 故障诊断

当 Envoy 向上游集群发起 TLS 连接时，所有的错误会被记录在 
[UPSTREAM_TRANSPORT_FAILURE_REASON](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/observability/access_log/usage#config-access-log-format-upstream-transport-failure-reason)
字段或 
[AccessLogCommon.upstream_transport_failure_reason](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/data/accesslog/v3/accesslog.proto#envoy-v3-api-field-data-accesslog-v3-accesslogcommon-upstream-transport-failure-reason)
字段。

当 Envoy 监听器接受连接并与下游建立 TLS 时，任何错误都会被记录在 
[DOWNSTREAM_TRANSPORT_FAILURE_REASON](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/observability/access_log/usage#config-access-log-format-downstream-transport-failure-reason)
字段或 
[AccessLogCommon.downstream_transport_failure_reason](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/data/accesslog/v3/accesslog.proto#envoy-v3-api-field-data-accesslog-v3-accesslogcommon-downstream-transport-failure-reason)
字段。

常见错误类型：

- ·Secret is not supplied by SDS`：Envoy 正在等待 SDS 提供密钥/证书或根证书。
- ·SSLV3_ALERT_CERTIFICATE_EXPIRED`：对端证书已过期，配置中不允许使用。
- ·SSLV3_ALERT_CERTIFICATE_UNKNOWN`：对端证书不在配置指定的 SPKI 名单内。
- ·SSLV3_ALERT_HANDSHAKE_FAILURE`：握手失败，通常是因为上游需要客户端证书，但客户端没有提供。
- ·TLSV1_ALERT_PROTOCOL_VERSION`：TLS 协议版本不一致。
- ·TLSV1_ALERT_UNKNOWN_CA`：对端证书的 CA 不在受信任 CA 列表中。

关于 BoringSSL 可能报告的更多详细错误列表，可在[此链接](https://github.com/google/boringssl/blob/master/crypto/err/ssl.errordata)查阅。


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

# 基于角色的访问控制

- [网络过滤器配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/ext_authz_filter#config-network-filters-ext-authz)

- [HTTP过滤器配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/ext_authz_filter#config-http-filters-ext-authz)

基于角色的访问控制（RBAC）过滤器负责检查传入请求是否有相应的授权。
不同于外部授权，RBAC过滤器直接在Envoy进程内部进行权限检查，这一过
程基于过滤器配置中定义的策略列表。

RBAC过滤器既可作为
[网络过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/rbac_filter#config-network-filters-rbac)
配置，也可作为
[HTTP过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/rbac_filter#config-http-filters-rbac)
配置，还可以两者
同时使用。如果网络过滤器判定请求未获授权，那么将关闭连接。相应地，如果
HTTP过滤器判定请求未授权，则会以403 Forbidden（禁止）状态码拒绝请求。

RBAC过滤器的规则可以通过配置一系列[策略](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/rbac/v3/rbac.proto#envoy-v3-api-field-config-rbac-v3-rbac-policies)来设置，也可以使用[匹配的API](https://www.envoyproxy.io/docs/envoy/v1.28.0/xds/type/matcher/v3/matcher.proto#envoy-v3-api-msg-xds-type-matcher-v3-matcher)进行
配置。

## 策略

RBAC过滤器根据一系列[策略](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/rbac/v3/rbac.proto#envoy-v3-api-field-config-rbac-v3-rbac-policies)来检查请求。每个策略包含了一组[权限](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/rbac/v3/rbac.proto#envoy-v3-api-msg-config-rbac-v3-permission)和一组[主体(principals. )](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/rbac/v3/rbac.proto#envoy-v3-api-msg-config-rbac-v3-principal)。
权限定义了请求可执行的操作，如HTTP请求的方法和路径。主体定义了请求的下游
客户端标识，如下游客户端证书中的URI SAN。当权限和主体同时满足时，视为
策略匹配成功。

## 匹配器

除了定义[策略](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/rbac/v3/rbac.proto#envoy-v3-api-field-config-rbac-v3-rbac-policies)外，RBAC过滤器还可以通过[匹配的API](https://www.envoyproxy.io/docs/envoy/v1.28.0/xds/type/matcher/v3/matcher.proto#envoy-v3-api-msg-xds-type-matcher-v3-matcher)来配置。在RBAC网络过滤器和
HTTP过滤器中都可以使用[网络输入](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/advanced/matching/matching_api#extension-category-envoy-matching-network-input)，而HTTP输入仅限于HTTP过滤器中使用。

[RBAC的匹配器扩展](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/rbac/matchers#api-v3-config-rbac-matchers)不支持[匹配的API](https://www.envoyproxy.io/docs/envoy/v1.28.0/xds/type/matcher/v3/matcher.proto#envoy-v3-api-msg-xds-type-matcher-v3-matcher)。

## 影子策略与影子匹配器

可以为过滤器配置影子策略或影子匹配器，它们不会实际影响请求（即不拒绝请求），
它们的作用仅限于输出统计数据和记录操作结果。这有助于在将规则部署到生产环境
之前进行测试。

## 条件

除了预设的权限和主体外，策略还可以根据需要提供一个使用[通用表达式语言](https://github.com/google/cel-spec/blob/master/doc/intro.md)
(Common Expression Language, CEL) 编写的授权条件。这个条件是策略匹配必须
满足的附加要求。例如，以下条件用于检测请求路径是否以 `/v1/` 开头：

`
call_expr:
  function: startsWith
  args:
  - select_expr:
     operand:
       ident_expr:
         name: request
     field: path
  - const_expr:
     string_value: /v1/
`

Envoy 提供了众多[请求属性](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/advanced/attributes#arch-overview-request-attributes)，以便实现富有表现力的策略。这些属性大多数是可选的，
并且会根据属性的类型提供默认值。CEL 支持使用 `has()` 语法来检查属性和映射
是否存在，比如 `has(request.referer)`。

# Envoy威胁模型

我们在此详细说明Envoy威胁模型，该模型对Envoy的操作者、开发人员以及安全研究人员都十分重要。
关于安全发布流程的更多信息，请访问 https://github.com/envoyproxy/envoy/security/policy 。

## 数据的机密性、完整性和系统的可用性

数据的机密性和完整性受损是我们极其重视的问题。对于Envoy操作者来说，系统的可用性—尤其是与
拒绝服务(DoS)攻击和资源枯竭相关的问题—也是重大的安全挑战，这一点对于在边缘计算环境中部署Envoy的用户尤其如此。

我们将针对满足以下标准的信息披露启动安全发布程序：

- 所有引起数据机密性或完整性损失的问题都将触发安全发布流程。
- 诸如Query-of-Death (QoD) 或资源耗尽的可用性问题需要满足以下所有条件才会触发安全发布流程：
    - 受影响的组件被标记为加固（详见核心及扩展部分的加固组件列表）。
    - 问题发生的流量类型（上游或下游）与组件的加固标签一致。例如，被标记为“对下游不信任加固”的组件，受到下游请求影响。
    - 资源耗尽的问题还需要满足以下额外条件：
        - 如果现有的超时机制无法覆盖，或者设置短超时值不现实的情况下，需要同时满足：
            - 内存耗尽问题，包括请求内存使用量超过配置的头部或高水位限制100倍以上的情况。比如，10 KiB的客户端请求导致Envoy消耗了1 MiB的内存；
            - CPU使用极度不对等，Envoy的CPU使用量至少是客户端的100倍。

Envoy在处理CPU和内存DoS攻击方面的可用性策略仍在发展中，
尤其是对于蛮力攻击。我们认识到，蛮力攻击（例如放大因子
低于100的攻击）很可能会对作为云服务基础设施的一部分或被
僵尸网络利用的Envoy部署构成威胁。我们会持续公开迭代和修
复已知的资源问题，比如过载管理和水位线的改进。对于那些可
能对现有Envoy部署构成风险的蛮力攻击披露，我们会启动安全
处理程序。

请注意，目前Envoy的默认设置在可用性方面并不被认为是安全的。
运维人员必须显式地配置水位线、过载管理器、断路器以及Envoy的
其他资源相关功能，以确保健全的可用性保障。对于因缺少安全默认
配置而引发的安全问题，我们不会采取任何行动。我们将随着时间的
推移致力于提供更安全的默认配置，但是由于需要考虑向后兼容性和
性能问题，这将需要遵循破坏性更改的弃用政策。

## 数据平面和控制平面

我们将威胁模型分为数据平面和控制平面，这与Envoy从架构上对这
些概念的内部区分相呼应。Envoy的核心组件被设计为能够抵御来
自不受信任的下游客户端和上游服务端的威胁。因此，在风险评估
中，我们最关注的是不受信任的下游客户端流量或上游服务器流量对
数据平面构成的威胁。这反映了Envoy作为边缘服务器的使用场景，
以及作为服务网格中与不可信服务交互的网络组件的使用情况。

控制平面的管理服务器通常被视为可信赖的。因此，我们并不担心xDS
传输协议可能遭受的线级攻击。然而，通过xDS传递给Envoy的配置
可能来自不受信任的来源，且可能未经彻底的清理。例如，服务运维人
员可能在同一个Envoy上托管多个租户，租户可能会在
RouteConfiguration中设定用于头部匹配的正则表达式。在这种场景下，
正如上文所述，我们期待Envoy能够从保密性、完整性和可用性的角度
抵御恶意配置的风险。

我们通常假设在请求处理过程中被调用的服务，比如外部授权、凭证供
应商、速率限制服务等，是可信的。如果不是这样的情况，相关的扩展
会在其文档中明确声明。

## 核心和扩展

Envoy核心中的所有内容均可用于不受信任和受信任的部署环境，
唯一的例外是明确标为alpha状态的特性；alpha状态的特性仅支持
在受信任的部署中使用，并不适用于下述的威胁模型。因此，稳定版
的核心应当基于此模型进行加固。与核心代码相关的安全问题一般会
触发本文所述的安全发布流程。

> **注意**
>
> 以下提到的[contrib](https://www.envoyproxy.io/docs/envoy/v1.28.0/start/install#install-contrib)
> 扩展并不正式受到威胁模型或Envoy安全团队的
> 覆盖。所有下述情况均为最大努力基础。

以下扩展旨在针对不受信任的下游和上游流量进行加固：

# 扩展安全: robust_to_untrusted_downstream_and_upstream

- envoy.bootstrap.internal_listener
- envoy.clusters.eds
- envoy.clusters.logical_dns
- envoy.clusters.original_dst
- envoy.clusters.static
- envoy.clusters.strict_dns
- envoy.filters.http.decompressor
- envoy.filters.http.set_metadata
- envoy.filters.http.upstream_codec
- envoy.filters.listener.tls_inspector
- envoy.filters.network.connection_limit
- envoy.formatter.cel (alpha)
- envoy.formatter.metadata (alpha)
- envoy.formatter.req_without_query (alpha)
- envoy.health_check.event_sinks.file
- envoy.http.early_header_mutation.header_mutation (alpha)
- envoy.http.stateful_header_formatters.preserve_case
- envoy.internal_redirect_predicates.allow_listed_routes
- envoy.internal_redirect_predicates.previous_routes
- envoy.internal_redirect_predicates.safe_cross_scheme
- envoy.io_socket.user_space
- envoy.load_balancing_policies.cluster_provided
- envoy.load_balancing_policies.maglev
- envoy.load_balancing_policies.random
- envoy.load_balancing_policies.ring_hash
- envoy.load_balancing_policies.round_robin
- envoy.load_balancing_policies.subset
- envoy.matching.matchers.cel_matcher
- envoy.matching.matchers.ip
- envoy.matching.matchers.runtime_fraction
- envoy.network.dns_resolver.apple
- envoy.network.dns_resolver.cares
- envoy.network.dns_resolver.getaddrinfo
- envoy.path.match.uri_template.uri_template_matcher
- envoy.path.rewrite.uri_template.uri_template_rewriter
- envoy.quic.deterministic_connection_id_generator (alpha)
- envoy.regex_engines.google_re2
- envoy.request_id.uuid
- envoy.transport_sockets.alts
- envoy.transport_sockets.internal_upstream
- envoy.transport_sockets.starttls
- envoy.transport_sockets.tcp_stats (alpha)
- envoy.transport_sockets.tls
- envoy.transport_sockets.upstream_proxy_protocol
- envoy.udp_packet_writer.default
- envoy.udp_packet_writer.gso

以下扩展设计上应当避免面对数据平面的攻击向量，因此它们应对来自
不受信任的下游和上游的流量具备较强的鲁棒性：

# 扩展安全: data_plane_agnostic

- envoy.grpc_credentials.aws_iam (alpha)
- envoy.grpc_credentials.file_based_metadata (alpha)
- envoy.key_value.file_based (alpha)
- envoy.resource_monitors.fixed_heap (alpha)
- envoy.resource_monitors.injected_resource (alpha)
- envoy.stat_sinks.dog_statsd
- envoy.stat_sinks.graphite_statsd (alpha)
- envoy.stat_sinks.hystrix
- envoy.stat_sinks.metrics_service
- envoy.stat_sinks.open_telemetry (alpha)
- envoy.stat_sinks.statsd
- envoy.stat_sinks.wasm (alpha)
- envoy.watchdog.profile_action (alpha)

以下扩展目的在于加强对不受信任下游的抵抗力，同时假定上游是
可信的：

# 扩展安全: robust_to_untrusted_downstream

- envoy.access_loggers.file
- envoy.access_loggers.http_grpc
- envoy.access_loggers.open_telemetry
- envoy.access_loggers.stderr
- envoy.access_loggers.stdout
- envoy.access_loggers.tcp_grpc
- envoy.clusters.dynamic_forward_proxy
- envoy.compression.brotli.compressor
- envoy.compression.brotli.decompressor
- envoy.compression.gzip.compressor
- envoy.compression.gzip.decompressor
- envoy.compression.zstd.compressor
- envoy.filters.http.buffer
- envoy.filters.http.compressor
- envoy.filters.http.cors
- envoy.filters.http.csrf
- envoy.filters.http.dynamic_forward_proxy
- envoy.filters.http.ext_authz
- envoy.filters.http.fault
- envoy.filters.http.grpc_json_transcoder
- envoy.filters.http.grpc_web
- envoy.filters.http.header_to_metadata
- envoy.filters.http.health_check
- envoy.filters.http.ip_tagging
- envoy.filters.http.json_to_metadata (alpha)
- envoy.filters.http.jwt_authn
- envoy.filters.http.kill_request
- envoy.filters.http.lua
- envoy.filters.http.oauth2 (alpha)
- envoy.filters.http.on_demand
- envoy.filters.http.original_src (alpha)
- envoy.filters.http.ratelimit
- envoy.filters.http.rbac
- envoy.filters.http.router
- envoy.filters.http.sxg (alpha) (contrib builds only)
- envoy.filters.listener.local_ratelimit
- envoy.filters.listener.original_dst
- envoy.filters.listener.original_src (alpha)
- envoy.filters.listener.proxy_protocol
- envoy.filters.network.client_ssl_auth (contrib builds only)
- envoy.filters.network.envoy_mobile_http_connection_manager
- envoy.filters.network.ext_authz
- envoy.filters.network.http_connection_manager
- envoy.filters.network.local_ratelimit
- envoy.filters.network.ratelimit
- envoy.filters.network.rbac
- envoy.filters.network.tcp_proxy
- envoy.filters.udp.dns_filter (alpha)
- envoy.filters.udp.session.dynamic_forward_proxy (alpha)
- envoy.filters.udp.session.http_capsule (alpha)
- envoy.filters.udp_listener.udp_proxy
- envoy.health_checkers.grpc
- envoy.health_checkers.http
- envoy.health_checkers.tcp
- envoy.http.original_ip_detection.custom_header
- envoy.http.original_ip_detection.xff
- envoy.matching.common_inputs.environment_variable
- envoy.matching.matchers.consistent_hashing
- envoy.quic.crypto_stream.server.quiche (alpha)
- envoy.quic.proof_source.filter_chain (alpha)
- envoy.quic.server_preferred_address.fixed (alpha)
- envoy.retry_host_predicates.omit_canary_hosts
- envoy.retry_host_predicates.omit_host_metadata
- envoy.retry_host_predicates.previous_hosts
- envoy.retry_priorities.previous_priorities
- envoy.route.early_data_policy.default
- envoy.tls.key_providers.cryptomb (alpha) (contrib builds only)
- envoy.tls.key_providers.qat (alpha) (contrib builds only)
- envoy.tracers.datadog
- envoy.tracers.xray
- envoy.tracers.zipkin
- envoy.upstreams.http.generic
- envoy.upstreams.http.http
- envoy.upstreams.http.http_protocol_options
- envoy.upstreams.http.tcp
- envoy.upstreams.http.udp (alpha)
- envoy.upstreams.tcp.generic

以下扩展只有在下游和上游都被认为是可信赖的情况下才推荐使用：

# 扩展安全: requires_trusted_downstream_and_upstream

- envoy.bootstrap.vcl (alpha) (contrib builds only)
- envoy.clusters.aggregate
- envoy.clusters.redis
- envoy.compression.zstd.decompressor
- envoy.filters.http.aws_lambda (alpha)
- envoy.filters.http.aws_request_signing (alpha)
- envoy.filters.http.checksum (alpha) (contrib builds only)
- envoy.filters.http.dynamo (contrib builds only)
- envoy.filters.http.golang (alpha) (contrib builds only)
- envoy.filters.http.language (alpha) (contrib builds only)
- envoy.filters.http.squash (contrib builds only)
- envoy.filters.http.tap (alpha)
- envoy.filters.listener.http_inspector
- envoy.filters.network.dubbo_proxy (alpha)
- envoy.filters.network.golang (alpha) (contrib builds only)
- envoy.filters.network.mongo_proxy
- envoy.filters.network.mysql_proxy (alpha) (contrib builds only)
- envoy.filters.network.postgres_proxy (contrib builds only)
- envoy.filters.network.redis_proxy
- envoy.filters.network.rocketmq_proxy (alpha) (contrib builds only)
- envoy.filters.network.sip_proxy (alpha) (contrib builds only)
- envoy.filters.network.thrift_proxy
- envoy.filters.network.zookeeper_proxy (alpha)
- envoy.filters.sip.router (alpha) (contrib builds only)
- envoy.filters.thrift.header_to_metadata (alpha)
- envoy.filters.thrift.payload_to_metadata (alpha)
- envoy.filters.thrift.rate_limit (alpha)
- envoy.filters.thrift.router
- envoy.health_checkers.redis
- envoy.health_checkers.thrift (alpha)
- envoy.matching.input_matchers.hyperscan (alpha) (contrib builds only)
- envoy.network.connection_balance.dlb (alpha) (contrib builds only)
- envoy.regex_engines.hyperscan (alpha) (contrib builds only)
- envoy.router.cluster_specifier_plugin.golang (alpha) (contrib builds only)
- envoy.tls.cert_validator.spiffe (alpha)
- envoy.transport_sockets.raw_buffer
- envoy.transport_sockets.tap (alpha)

以下扩展的安全状况尚不明确：

# 扩展安全: unknown

- envoy.access_loggers.extension_filters.cel (alpha)
- envoy.access_loggers.wasm (alpha)
- envoy.bootstrap.wasm (alpha)
- envoy.config.validators.minimum_clusters_validator
- envoy.config_mux.delta_grpc_mux_factory
- envoy.config_mux.sotw_grpc_mux_factory
- envoy.config_subscription.ads
- envoy.config_subscription.ads_collection
- envoy.config_subscription.aggregated_delta_grpc_collection
- envoy.config_subscription.aggregated_grpc_collection
- envoy.config_subscription.delta_grpc
- envoy.config_subscription.filesystem
- envoy.config_subscription.filesystem_collection
- envoy.config_subscription.grpc
- envoy.config_subscription.rest
- envoy.filters.http.adaptive_concurrency
- envoy.filters.http.admission_control
- envoy.filters.http.alternate_protocols_cache (alpha)
- envoy.filters.http.bandwidth_limit
- envoy.filters.http.cdn_loop (alpha)
- envoy.filters.http.composite
- envoy.filters.http.connect_grpc_bridge (alpha)
- envoy.filters.http.ext_proc (alpha)
- envoy.filters.http.gcp_authn (alpha)
- envoy.filters.http.grpc_field_extraction (alpha)
- envoy.filters.http.grpc_http1_bridge
- envoy.filters.http.grpc_http1_reverse_bridge (alpha)
- envoy.filters.http.grpc_stats (alpha)
- envoy.filters.http.header_mutation (alpha)
- envoy.filters.http.local_ratelimit
- envoy.filters.http.set_filter_state (alpha)
- envoy.filters.http.stateful_session (alpha)
- envoy.filters.http.wasm (alpha)
- envoy.filters.network.direct_response
- envoy.filters.network.echo
- envoy.filters.network.set_filter_state (alpha)
- envoy.filters.network.sni_cluster
- envoy.filters.network.sni_dynamic_forward_proxy (alpha)
- envoy.filters.network.wasm (alpha)
- envoy.http.stateful_session.cookie (alpha)
- envoy.http.stateful_session.header (alpha)
- envoy.load_balancing_policies.least_request
- envoy.matching.actions.format_string
- envoy.matching.custom_matchers.trie_matcher
- envoy.matching.inputs.application_protocol
- envoy.matching.inputs.cel_data_input
- envoy.matching.inputs.destination_ip
- envoy.matching.inputs.destination_port
- envoy.matching.inputs.direct_source_ip
- envoy.matching.inputs.dns_san
- envoy.matching.inputs.filter_state
- envoy.matching.inputs.query_params
- envoy.matching.inputs.request_headers
- envoy.matching.inputs.request_trailers
- envoy.matching.inputs.response_headers
- envoy.matching.inputs.response_trailers
- envoy.matching.inputs.server_name
- envoy.matching.inputs.source_ip
- envoy.matching.inputs.source_port
- envoy.matching.inputs.source_type
- envoy.matching.inputs.subject
- envoy.matching.inputs.transport_protocol
- envoy.matching.inputs.uri_san
- envoy.rate_limit_descriptors.expr
- envoy.rbac.matchers.upstream_ip_port (alpha)
- envoy.tracers.dynamic_ot
- envoy.tracers.opencensus
- envoy.transport_sockets.http_11_proxy (alpha)
- envoy.upstream.local_address_selector.default_local_address_selector (alpha)
- envoy.upstreams.tcp.tcp_protocol_options (alpha)
- envoy.wasm.runtime.null (alpha)
- envoy.wasm.runtime.v8 (alpha)
- envoy.wasm.runtime.wamr (alpha)
- envoy.wasm.runtime.wasmtime (alpha)
- envoy.wasm.runtime.wavm (alpha)

Envoy目前包含两个动态过滤器扩展支持加载可执行代码的：WASM和
Lua。在这两种情况中，我们都假定动态加载的代码是受信任的。对于
Lua，我们预期其运行环境能够在可信脚本的前提下，抵御不受信任的
数据平面流量。WASM尚处于开发阶段，但预计最终将采取类似的安全
态度。

# 外部依赖项

下面我们列举了可能会链接进Envoy可执行文件的外部依赖。我们没有
包括那些仅在持续集成(CI)过程或开发工具中使用的依赖项。

原文地址： https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/security/external_deps

# Google漏洞奖励计划（VRP）

Envoy是Google漏洞奖励计划（VRP）的参与者。该计划对所有安全研究
者开放，按照以下规则报告漏洞可获得奖励。

## 规则

VRP的宗旨是为了正式表彰外部安全研究者对Envoy安全贡献的程序。
漏洞需满足以下条件，才符合计划资格：

1. 漏洞必须符合以下某一目标，通过提供的基于Docker的执行环境加
   以证明，并且要与该计划的威胁模型保持一致。

2. 漏洞必须报告至envoy-security@googlegroups.com，并在进行分类和
   可能的安全版本发布期间维持保密。提交报告时请遵守披露指导原
   则。披露SLO在此处有文档记录。一般而言，安全披露须遵循Linux
   Foundation的隐私政策，并且VRP报告（包括报告者的电子邮件地址
   和姓名）可自由与Google共享，用于VRP目的。

3. 漏洞不应在公开论坛中被提前知晓，例如GitHub问题跟踪器、CVE
   数据库（如果之前已与Envoy相关联）等。之前未与Envoy相关联的
   现有CVE可以考虑。

4. 漏洞不得同时提交给Google或Lyft运营的其他奖励计划。

奖励由Envoy开源安全团队和Google根据具体情况自行决定，并将依据
上述标准进行设定。如果同一漏洞被多位独立研究者同时报告，或者该
漏洞已由Envoy开源安全团队在保密状态下跟踪处理，我们会尽力公平地
在报告者间分配奖金。

## 威胁模型

基本威胁模型与Envoy的开源安全姿态相一致。为了在该计划初期提供
受约束的攻击表面，我们增加了一些临时限制。我们排除了以下来源的
任何威胁：

- 不可信的控制平面。
- 如访问日志、外部授权等运行时服务。
- 不可信的上游源。
- DoS攻击，除非另有规定。
- 除HTTP连接管理器网络过滤器和HTTP路由器过滤器之外的所有过滤器。
- 管理控制台；在执行环境中已被禁用。

我们还明确排除了对Envoy进程的任何本地攻击（例如，通过本地进程、
Shell等）。所有攻击必须通过端口10000上的网络数据平面发起。此外，
内核和Docker漏洞不在这个威胁模型范围内。

将来随着我们增强计划执行环境的复杂度，我们可能会放宽一些这些限制。

## 执行环境

我们提供了Docker镜像，这些镜像是本计划的参考环境：

- [envoyproxy/envoy-google-vrp](https://hub.docker.com/r/envoyproxy/envoy-google-vrp/tags/)
  镜像是基于Envoy点版本发布的。只有
  在提交漏洞时的最新点版本才有资格参加本计划。VRP计划首个可用
  的点版本是1.15.0 Envoy发布。

- [envoyproxy/envoy-google-vrp-dev](https://hub.docker.com/r/envoyproxy/envoy-google-vrp-dev/tags/)
  镜像是基于Envoy主分支构建的。
  只有提交漏洞时最近5天内的构建版本才有资格参加本计划。在提交
  时，这些构建不应受到任何已公开披露漏洞的影响。

当通过docker run命令启动这些镜像时，可以访问两个Envoy进程：

- 边缘Envoy监听10000端口（HTTPS）。它的静态配置根据Envoy的边缘加
  固原则设定。它设置了三种路由规则（依次为）：

    1. `/content/*`：路由至原始Envoy服务器。
    
    2. `/*`：返回403（禁止访问）。

- 原始Envoy作为边缘Envoy的上游。它的静态配置只包含直接响应，
  实质上作为一个HTTP原始服务器。它有两个路由规则（依次为）：

    1. `/blockedz`：返回200 `hidden treasure`。除非有符合条件的漏洞，
       否则边缘Envoy服务器的10000端口不应该收到这个回应。

    2. `/*`：返回200 `normal`。

当运行Docker镜像时，应该提供以下命令行选项：

- `-m 3g` 以确保内存被限制在3GB。执行环境至少应提供这么多内存。
  每个Envoy进程都有一个过载管理器配置，限制内存使用不超过1GB。

- `-e ENVOY_EDGE_EXTRA_ARGS="<...>"` 用于提供边缘Envoy的额外CLI
  参数。这个参数需要设置，但其值可以为空。

- `-e ENVOY_ORIGIN_EXTRA_ARGS="<...>"` 用于提供原始Envoy的额外CLI
  参数。这个参数需要设置，但其值可以为空。

## 目标

以下类别的故障模式将通过在10000端口上的请求来证实漏洞存在：

- 死亡查询(Query-of-death)：导致Envoy进程立即发生段错误(segfault)
  或中止的请求。
- 内存耗尽(OOM)：导致边缘Envoy进程发生内存耗尽的请求。造成这种
  情况的连接和流总数不得超过100（即排除了通过连接/流的暴力DoS攻
  击）。
- 路由规则绕过：能够访问隐藏宝藏(hidden treasure)的请求。
- TLS证书泄露：能够获取边缘Envoy的`serverkey.pem`证书的请求。
- 远程代码执行：通过网络数据平面获取的任何root shell。
- 根据OSS Envoy安全团队的判断，足够有趣的漏洞即使不符合上述类
  别，但很可能属于高危或严重漏洞的类别。

## 使用Docker镜像

要在本地端口10000启动边缘Envoy的基本命令如下：

```
docker run -m 3g -p 10000:10000 --name envoy-google-vrp \
  -e ENVOY_EDGE_EXTRA_ARGS="" \
  -e ENVOY_ORIGIN_EXTRA_ARGS="" \
  envoyproxy/envoy-google-vrp-dev:latest
```

在进行调试时，可能需要额外的参数，例如获取跟踪日志，或使用wireshark和gdb时：

```
docker run -m 3g -p 10000:10000 --name envoy-google-vrp \
  -e ENVOY_EDGE_EXTRA_ARGS="-l trace" \
  -e ENVOY_ORIGIN_EXTRA_ARGS="-l trace" \
  --cap-add SYS_PTRACE --cap-add NET_RAW --cap-add NET_ADMIN \
  envoyproxy/envoy-google-vrp-dev:latest
```

如果需要在Docker容器中获取一个shell，可以使用：

```
docker exec -it envoy-google-vrp /bin/bash
```

Docker镜像中包括了gdb、strace、tshark等工具（欢迎通过提交PR更新[Docker构建文件](https://github.com/envoyproxy/envoy/blob/v1.28.0//ci/Dockerfile-envoy-google-vrp)来提供其他工具的建议）。

## 重建Docker镜像

为了研究目的，能够自己重新生成Docker基础镜像是非常有帮助的。要在不依赖CI的情况下进行这一操作，请跟随[ci/docker_rebuild_google-vrp.sh](https://github.com/envoyproxy/envoy/blob/v1.28.0/ci/docker_rebuild_google-vrp.sh)文件顶部的指引。这一操作流程的一个示例如下：

```
bazel build //source/exe:envoy-static
./ci/docker_rebuild_google-vrp.sh bazel-bin/source/exe/envoy-static
docker run -m 3g -p 10000:10000 --name envoy-google-vrp \
  -e ENVOY_EDGE_EXTRA_ARGS="" \
  -e ENVOY_ORIGIN_EXTRA_ARGS="" \
  envoy-google-vrp:local
```
