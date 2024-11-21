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

## TCP statistics {#config_listener_stats_tcp}

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

以下 UDP 统计信息适用于 UDP 侦听器，并且以 *listener.<address>.udp.* 为根：

|名称                            |类型             |描述|
|--------------------------------|-----------------|------------------------------|
|downstream_rx_datagram_dropped  |Counter          |由于内核溢出或截断而丢弃的数据报数量|

## 每个监听处理程序统计信息

每个监听器都另外有一个统计信息树，其根为 *listener.<address>.<handler>.*，其中包含每个处理程序的统计信息。如 
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

