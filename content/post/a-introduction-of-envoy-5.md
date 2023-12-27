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
