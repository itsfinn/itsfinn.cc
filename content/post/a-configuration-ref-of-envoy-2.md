---
title: "Envoy 配置指南(二)"
date: 2024-11-20T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网配置指南的中文翻译"
Tags: ["envoy", "配置指南", "中文文档"]
Categories: ["envoy"]
DisableComments: false
---

本文档基于 v1.28, 原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/configuration

Envoy 官网配置指南的中文翻译(监听):统计数据、运行时、监听过滤器、网络过滤器、UDP监听过滤器、监听发现服务（LDS）
<!--more-->

# 概述

顶级 Envoy 配置包含一个
[侦听器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners)
列表。每个单独的侦听器配置都具有以下格式：

[v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener.proto#envoy-v3-api-msg-config-listener-v3-listener)

# 统计数据

## 监听器

每个监听器都有一个以 *listener.\<address\>.* 为根的统计树（如果 stat_prefix 非空，则为 *listener.<stat_prefix>.*），包含以下统计信息：

|名称                                             |类型               |描述|
|------------------------------------------------|----------------- |------------------------------------|
|downstream_cx_total                             |Counter           |连接总数|
|downstream_cx_destroy                           |Counter           |被破坏的连接总数|
|downstream_cx_active                            |Gauge             |活动连接总数|
|downstream_cx_length_ms                         |Histogram         |连接长度 毫秒|
|downstream_cx_transport_socket_connect_timeout  |Counter           |传输套接字连接协商期间超时的连接总数|
|downstream_cx_overflow                          |Counter           |由于强制执行侦听器连接限制而拒绝的连接总数|
|downstream_cx_overload_reject                   |Counter           |由于配置的过载操作而拒绝的连接总数|
|downstream_global_cx_overflow                   |Counter           |由于强制执行全局连接限制而拒绝的连接总数|
|connections_accepted_per_socket_event           |Histogram         |每个监听套接字事件接受的连接数|
|downstream_pre_cx_timeout                       |Counter           |侦听器过滤器处理期间超时的套接字|
|downstream_pre_cx_active                        |Gauge             |当前正在接受侦听器过滤器处理的套接字|
|extension_config_missing                        |Counter           |由于缺少侦听器过滤器扩展配置，关闭的连接总数|
|network_extension_config_missing                |Counter           |由于缺少网络过滤器扩展配置而关闭的连接总数|
|global_cx_overflow                              |Counter           |由于强制执行全局连接限制而拒绝的连接总数|
|no_filter_chain_match                           |Counter           |不匹配任何过滤链的连接总数|
|downstream_listener_filter_remote_close         |Counter           |当侦听器过滤器查看数据时远程关闭的连接总数|
|downstream_listener_filter_error                |Counter           |监听过滤器获取数据时读取错误的总数|

## TLS 统计数据

以下 TLS 统计数据的根为 *listener.\<address\>.ssl.*：

|名称                    |类型             |描述|
|-----------------------|-----------------|-----------------------------------------------------------|
|connection_error       |Counter          |TLS 连接错误总数（不包括失败的证书验证）|
|handshake              |Counter          |成功的 TLS 连接握手总数|
|session_reused         |Counter          |成功恢复 TLS 会话的总数|
|no_certificate         |Counter          |没有客户端证书的成功 TLS 连接总数|
|fail_verify_no_cert    |Counter          |由于缺少客户端证书而失败的 TLS 连接总数|
|fail_verify_error      |Counter          |未通过 CA 验证的 TLS 连接总数|
|fail_verify_san        |Counter          |未通过 SAN 验证的 TLS 连接总数|
|fail_verify_cert_hash  |Counter          |证书固定验证失败的 TLS 连接总数|
|ocsp_staple_failed     |Counter          |不符合 OCSP 策略的 TLS 连接总数|
|ocsp_staple_omitted    |Counter          |未绑定 OCSP 响应且成功的 TLS 连接总数|
|ocsp_staple_responses  |Counter          |具有有效 OCSP 响应的 TLS 连接总数（无论客户端是否请求装订 OCSP 响应）|
|ocsp_staple_requests   |Counter          |客户端请求装订 OCSP 响应的 TLS 连接总数|
|ciphers.\<cipher\>     |Counter          |使用密码套件 \<cipher\> 的成功 TLS 连接总数|
|curves.\<curve\>       |Counter          |使用椭圆曲线 \<curve\> 的成功 TLS 连接总数|
|sigalgs.\<sigalg\>     |Counter          |使用签名算法 \<sigalg\> 的成功 TLS 连接总数|
|versions.\<version\>   |Counter          |使用协议版本 \<version\> 的成功 TLS 连接总数|
|was_key_usage_invalid  |Counter          |使用[无效 keyUsage 扩展](https://github.com/google/boringssl/blob/6f13380d27835e70ec7caf807da7a1f239b10da6/ssl/internal.h#L3117)的成功 TLS 连接总数。（由于 [issue #28246](https://github.com/envoyproxy/envoy/issues/28246)，此功能在 BoringSSL FIPS 中尚不可用）|

## TCP statistics

使用 TCP 统计传输套接字时可用的以下 TCP 统计信息以 *listener.\<address\>.tcp_stats.* 为根：

> **注意**
> 这些指标由操作系统提供。由于可用的操作系统指标和测量方法存在差异，因此不同操作系统或同一操作系统的不同版本之间的值可能不一致。

|名称                                   |类型             |描述|
|--------------------------------------|-----------------|--------------------------------------|
|cx_tx_segments                        |Counter          |已传输的 TCP 段总数|
|cx_rx_segments                        |Counter          |已接收的 TCP 段总数|
|cx_tx_data_segments                   |Counter          |已传输非零数据长度的 TCP 段总数|
|cx_rx_data_segments                   |Counter          |收到的数据长度非零的 TCP 段总数|
|cx_tx_retransmitted_segments          |Counter          |重新传输的 TCP 段总数|
|cx_rx_bytes_received                  |Counter          |已接收并发送 TCP 确认的总有效负载字节数。|
|cx_tx_bytes_sent                      |Counter          |已传输的有效负载字节总数（包括重新传输的字节）。|
|cx_tx_unsent_bytes                    |Gauge            |Envoy 已发送给操作系统但尚未发送的字节数|
|cx_tx_unacked_segments                |Gauge            |已发送但尚未被确认的段|
|cx_tx_percent_retransmitted_segments  |Histogram        |连接中重新传输的段的百分比|
|cx_rtt_us                             |Histogram        |平滑的往返时间估计（以微秒为单位）|
|cx_rtt_variance_us                    |Histogram        |估计往返时间的微秒差异。值越高，差异越大。|

## UDP 统计信息

以下 UDP 统计信息适用于 UDP 侦听器，并且以 *listener.\<address\>.udp.* 为根：

|名称                            |类型             |描述|
|--------------------------------|-----------------|------------------------------|
|downstream_rx_datagram_dropped  |Counter          |由于内核溢出或截断而丢弃的数据报数量|

## 每个监听处理程序统计信息

每个监听器都另外有一个统计信息树，其根为 *listener.\<address\>.\<handler\>.*，其中包含每个处理程序的统计信息。如 
[线程模型](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/intro/threading_model#arch-overview-threading)
文档中所述，Envoy 有一个线程模型，其中包括主线程以及由 `--concurrency` 选项控制的 多个工作线程。 根路径里的 \<handler\> 等于main_thread、 worker_0、worker_1 等。这些统计数据可用于查找已接受或活动连接上每个处理程序/工作程序的不平衡情况。

|名称                   |类型             |描述|
|----------------------|-----------------|-----------------------------------------|
|downstream_cx_total   |Counter          |此处理程序上的总连接数。|
|downstream_cx_active  |Gauge            |此处理程序上的活动连接总数。|

## 监听管理器

监听管理器有一个以 *listener_manager* 为根的统计信息树，包含以下统计信息。统计信息名称中的任何字符 `:` 都将被替换为 `_`。

|名称                           |类型       |描述|
|------------------------------|----------|----------------------------------------------------------------|
|listener_added                |Counter   |已添加的监听器总数（通过静态配置或 LDS）。|
|listener_modified             |Counter   |修改的听众总数（通过 LDS）。|
|listener_removed              |Counter   |已删除的听众总数（通过 LDS）。|
|listener_stopped              |Counter   |停止的监听总数|
|listener_create_success       |Counter   |添加到工作线程成功的监听器总数|
|listener_create_failure       |Counter   |添加到工作线程失败的监听器总数|
|listener_in_place_updated     |Counter   |为执行过滤器链更新路径而创建的监听器对象总数。|
|total_filter_chains_draining  |Gauge     |当前正在排空的过滤器链的数量。|
|total_listeners_warming       |Gauge     |当前正在预热的监听器总数|
|total_listeners_active        |Gauge     |当前活跃的监听器总数|
|total_listeners_draining      |Gauge     |当前正在排空的监听器总数|
|workers_started               |Gauge     |一个布尔值（如果已启动则为 1，否则为 0），指示监听器是否已在工作线程上初始化。|

# 运行时

支持以下运行时设置：

envoy.resource_limits.listener.\<name of listener\>.connection_limit

设置指定侦听器的活动连接数限制。

# 监听过滤器

Envoy 具有以下内置监听过滤器。

## HTTP Inspector

HTTP Inspector 监听过滤器允许检测应用程序协议是否为 HTTP，
如果是 HTTP，它会进一步检测 HTTP 协议（​​HTTP/1.x 或 HTTP/2）。
可以通过
[FilterChainMatch](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-msg-config-listener-v3-filterchainmatch)
的 
[application_protocols](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-field-config-listener-v3-filterchainmatch-application-protocols)
选择 
[FilterChain](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-msg-config-listener-v3-filterchain)
来使用 HTTP Inspector 监听过滤器。

- 此过滤器应配置为 URL 类型
  `type.googleapis.com/envoy.extensions.filters.listener.http_inspector.v3.HttpInspector`。
- [监听器过滤器 v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/listener/http_inspector/v3/http_inspector.proto#envoy-v3-api-msg-extensions-filters-listener-http-inspector-v3-httpinspector)

过滤器配置示例如下：

``` yaml
listener_filters:
  - name: "envoy.filters.listener.http_inspector"
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.listener.http_inspector.v3.HttpInspector
```

此过滤器有一个以 *http_inspector* 为根的统计树，其统计数据如下：

|名称 |类型 |描述|
|----------------|-----------------|---------------------------------------------------------------|
|read_error |计数器 |总读取错误|
|http10_found |计数器 |发现 HTTP/1.0 的总次数|
|http11_found |计数器 |发现 HTTP/1.1 的总次数|
|http2_found |计数器 |发现 HTTP/2 的总次数|
|http_not_found |计数器 |未找到 HTTP 协议的总次数|

## 本地速率限制

- 本地速率限制`架构概述`
- 此过滤器 URL 类型应配置为 `type.googleapis.com/envoy.extensions.filters.listener.local_ratelimit.v3.LocalRateLimit`。
  v3 API 参考

> **注意**
>
> 令牌桶在所有工作进程之间共享，因此速率限制适用于每个 Envoy 进程。

> **注意**
>
> 还通过“全局速率限制网络过滤器”支持网络层的全局速率限制。

### 概述

本地速率限制过滤器采用“令牌桶”速率
限制由过滤器的过滤器链处理的传入套接字。每个套接字
过滤器处理的每个请求都使用一个令牌，如果没有可用的令牌，套接字将
立即关闭，无需进一步过滤迭代。

> **注意**
>
> 在当前实现中，每个过滤器和过滤器链都有独立的速率限制。

### 统计数据

每个配置的本地速率限制过滤器都有以 *listener_local_ratelimit.\<stat_prefix\>.* 为根的统计信息。
统计数据如下：

|名称 |类型 |描述|
|----------------|-----------------|-------------|
|rate_limited |计数器|由于超出速率限制而关闭的总套接字数|

### 运行时

本地速率限制过滤器可以通过“启用”来标记运行时功能配置字段。

## Original Destination

### Linux

当连接已被 iptables REDIRECT 目标重定向，或者被 iptables TPROXY 目标结合设置侦听器的“透明”选项重定向时，原始目标侦听器过滤器会读取 SO_ORIGINAL_DST 套接字选项集。

### Windows

当连接被应用于容器端点的 [HNS](https://docs.microsoft.com/en-us/virtualization/windowscontainers/container-networking/architecture#container-network-management-with-host-network-service) 策略重定向时，原始目标侦听器过滤器会读取设置的 SO_ORIGINAL_DST 套接字选项。要使此过滤器正常工作，必须在侦听器上设置 [traffic_direction](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener.proto#envoy-v3-api-field-config-listener-v3-listener-traffic-direction)。这意味着需要一个单独的侦听器来处理入站和出站流量。

重定向不适用于所有类型的网络流量。支持重定向的数据包类型如下表所示：

- TCP/IPv4
- UDP
- 原始 UDPv4，无标头包含选项
- 原始 ICMP

有关详细信息，请参阅[使用绑定或连接重定向](https://docs.microsoft.com/en-us/windows-hardware/drivers/network/using-bind-or-connect-redirection)

> **注意**
> 在撰写本文时（2021 年 2 月），操作系统对原始目标的支持仅通过
> [Windows 预览体验计划](https://insider.windows.com/en-us/for-developers)。
> 该功能将在即将发布的 Windows Server 版本中得到全面支持，请参阅
> [Windows Server 发布信息](https://docs.microsoft.com/en-us/windows-server/get-started/windows-server-release-info)。

Envoy 中的后续处理将恢复的目标地址视为连接的本地地址，而不是侦听器正在侦听的地址。此外，
[原始目标集群](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/upstream/service_discovery#arch-overview-service-discovery-types-original-destination)
可用于将 HTTP 请求或 TCP 连接转发到恢复的目标地址。

### 内部监听器

原始目标侦听器过滤器读取由
[内部侦听器](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/other_features/internal_listener#config-internal-listener)
而不是系统套接字选项处理的用户空间套接字上的动态元数据和过滤器状态对象。

目标地址的动态元数据应放置在字段 *local* 下的键 *envoy.filters.listener.original_dst* 中，并应包含带有 IP 和端口地址的字符串。如果没有动态元数据，则参考过滤器状态。

该过滤器使用的过滤状态对象是：

- *envoy.filters.listener.original_dst.local_ip* 为目标地址
- *envoy.filters.listener.original_dst.source_ip* 为源地址

请注意
[内部上游传输](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/other_features/internal_listener#config-internal-upstream-transport)
应用于将动态元数据从端点主机传递到套接字元数据和(或)通过用户空间套接字与上游连接共享的过滤器状态对象到内部侦听器。

- 此过滤器 URL 类型应配置为`type.googleapis.com/envoy.extensions.filters.listener.original_dst.v3.OriginalDst`。
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/listener/original_dst/v3/original_dst.proto#envoy-v3-api-msg-extensions-filters-listener-original-dst-v3-originaldst)


## Original Source

- 此过滤器 URL 类型应配置为`type.googleapis.com/envoy.extensions.filters.listener.original_src.v3.OriginalSrc`。
- [监听器过滤器 v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/listener/original_src/v3/original_src.proto#envoy-v3-api-msg-extensions-filters-listener-original-src-v3-originalsrc)

原始源侦听器过滤器会在 Envoy 的上游复制连接的下游远程地址。例如，如果下游连接使用 IP 地址 `10.1.2.3` 连接到 Envoy，则 Envoy 将使用源 IP `10.1.2.3` 连接到上游。

> **注意**
> Windows 不支持此过滤器。

> **注意**
> Linux 上需要 CAP_NET_ADMIN 功能。

### 与 Proxy Protocol 交互

如果连接的源地址尚未转换或代理，则 Envoy 可以简单地使用现有的连接信息来构建正确的下游远程地址。但是，如果不是这样，则可以使用
[代理协议过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/listeners/listener_filters/proxy_protocol#config-listener-filters-proxy-protocol)
来提取下游远程地址。

### IP 版本支持

该过滤器支持 IPv4 和 IPv6 地址。请注意，上游连接必须支持使用的版本。

### 额外设置

使用的下游远程地址很可能是全局可路由的。默认情况下，从上游主机返回该地址的数据包不会通过 Envoy 路由。必须配置网络以强制将 IP 被 Envoy 复制的任何流量路由回 Envoy 主机。

如果 Envoy 和上游位于同一主机上, 例如在 sidecar 部署中，则可以使用 iptables 和路由规则来确保正确的行为。过滤器具有无符号整数配置[mark](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/listener/original_src/v3/original_src.proto#envoy-v3-api-field-extensions-filters-listener-original-src-v3-originalsrc-mark)。将其设置为 *X* 会导致 Envoy 使用值 *X* 标记来自此侦听器的所有上游数据包。请注意，如果将[mark](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/listener/original_src/v3/original_src.proto#envoy-v3-api-field-extensions-filters-listener-original-src-v3-originalsrc-mark)设置为 0，Envoy 将不会标记上游数据包。

我们可以使用以下命令集来确保所有标有 *X*（示例中假设为 123）的 ipv4 和 ipv6 流量都能正确路由。请注意，此示例假设 *eth0* 是默认出站接口。

```text
iptables -t mangle -I PREROUTING -m mark --mark 123 -j CONNMARK --save-mark
iptables -t mangle -I OUTPUT -m connmark --mark 123 -j CONNMARK --restore-mark
ip6tables -t mangle -I PREROUTING -m mark --mark 123 -j CONNMARK --save-mark
ip6tables -t mangle -I OUTPUT -m connmark --mark 123 -j CONNMARK --restore-mark
ip 规则添加 fwmark 123 查找 100
ip 路由添加本地 0.0.0.0/0 dev lo 表 100
ip -6 规则添加 fwmark 123 查找 100
ip -6 路由添加本地::/0 dev lo 表 100
echo 1 > /proc/sys/net/ipv4/conf/eth0/route_localnet
```

### 监听器配置示例

以下示例将 Envoy 配置为对端口 8888 上的所有连接使用原始源。它使用代理协议来确定下游远程地址。所有上游数据包都标记为 123。

``` yaml
listeners:
- address:
    socket_address:
      address: 0.0.0.0
      port_value: 8888
  listener_filters:
    - name: envoy.filters.listener.proxy_protocol
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.listener.proxy_protocol.v3.ProxyProtocol
    - name: envoy.filters.listener.original_src
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.listener.original_src.v3.OriginalSrc
        mark: 123
```

## Proxy Protocol

此监听过滤器增加了对 [HAProxy 代理协议](https://www.haproxy.org/download/1.9/doc/proxy-protocol.txt) 的支持。

在此模式下，下游连接假定来自代理，代理将原始坐标（IP、PORT）放入连接字符串中。然后，Envoy 提取这些并将它们用作远程地址。

在代理协议 v2 中，存在可选的扩展 (TLV) 标签的概念。如果将 TLV 的类型添加到过滤器的配置中，则 TLV 将作为具有用户指定密钥的动态元数据发出。

此实现支持版本 1 和版本 2，它会根据每个连接自动确定存在哪个版本。注意：如果启用了过滤器，则代理协议必须存在于连接上（版本 1 或版本 2），标准不允许通过解析来确定它是否存在。

如果存在协议错误或不受支持的地址系列（例如 AF_UNIX），则连接将关闭并抛出错误。

- 此过滤器应配置为 URL 类型`type.googleapis.com/envoy.extensions.filters.listener.proxy_protocol.v3.ProxyProtocol`。
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/listener/proxy_protocol/v3/proxy_protocol.proto#envoy-v3-api-msg-extensions-filters-listener-proxy-protocol-v3-proxyprotocol)

### 统计数据

该过滤器发出以下统计数据：

|名称 |类型 |描述|
|--------------------------------|------|-------------|
|downstream_cx_proxy_proto_error |计数器 |代理协议错误总数|


## TLS Inspector

TLS 检查器侦听器过滤器允许检测传输是否显示为 TLS 或纯文本，如果是 TLS，它会检测来自客户端的 [Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication) 和/或 [应用层协议协商](https://en.wikipedia.org/wiki/Application-Layer_Protocol_Negotiation)。这可用于通过 
[FilterChainMatch](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-field-config-listener-v3-filterchainmatch-application-protocols)
的
[server_names](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-field-config-listener-v3-filterchainmatch-server-names) 
和/或 
[application_protocols](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-field-config-listener-v3-filterchainmatch-application-protocols)
选择
[FilterChain](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-msg-config-listener-v3-filterchain)。

- [SNI](https://www.envoyproxy.io/docs/envoy/v1.28.7/faq/configuration/sni#faq-how-to-setup-sni)
- 此过滤器应使用类型 URL `type.googleapis.com/envoy.extensions.filters.listener.tls_inspector.v3.TlsInspector` 配置。
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/listener/tls_inspector/v3/tls_inspector.proto#envoy-v3-api-msg-extensions-filters-listener-tls-inspector-v3-tlsinspector)

### 例子

示例过滤器配置可能是：

```yaml
listener_filters:
- name: tls_inspector
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.filters.listener.tls_inspector.v3.TlsInspector
```

### 统计数据

此过滤器有一个以 *tls_inspector* 为根的统计树，其统计数据如下：

|名称 |类型 |描述|
|------------------------------------|-----------------|------------------|
|client_hello_too_large |计数器 |总共收到不合理的过大客户端 hello|
|tls_found |计数器 |发现 TLS 的总次数|
|tls_not_found |计数器 |未找到 TLS 的总次数|
|alpn_found |计数器 |[应用层协议协商](https://en.wikipedia.org/wiki/Application-Layer_Protocol_Negotiation)成功的总次数|
|alpn_not_found |计数器 |[应用层协议协商](https://en.wikipedia.org/wiki/Application-Layer_Protocol_Negotiation)失败的总次数|
|sni_found |计数器 |找到 [Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication) 的总次数|
|sni_not_found |计数器 |未找到 [Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication) 的总次数|
|bytes_processed |直方图 |记录大小，记录 tls_inspector 在分析 tls 使用情况时处理的字节数。如果连接使用 TLS：这是客户端 hello 的大小。如果客户端 hello 太大，则记录的值将为 64KiB，这是最大客户端 hello 大小。如果连接不使用 TLS：这是检查器确定连接未使用 TLS 之前处理的字节数。如果连接提前终止，并且我们没有足够的字节来应对上述任何一种情况，则不会记录任何内容。|
