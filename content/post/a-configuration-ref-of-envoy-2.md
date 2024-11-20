---
title: "Envoy 配置指南(二)"
date: 2024-11-20T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网配置指南的中文翻译"
Tags: ["envoy", "配置指南", "中文文档"]
Categories: ["envoy"]
DisableComments: false
---

本文档基于 v1.28, 原文地址: [https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/configuration)

Envoy 官网配置指南的中文翻译(监听):统计数据、运行时、监听过滤器、网络过滤器、UDP监听过滤器、监听发现服务（LDS）
<!--more-->

# 概述

顶级 Envoy 配置包含一个
[侦听器](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/listeners/listeners#arch-overview-listeners)
列表。每个单独的侦听器配置都具有以下格式：

[v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener.proto#envoy-v3-api-msg-config-listener-v3-listener)

# 统计数据

## 监听器

每个监听器都有一个以 listener.\<address\>. 为根的统计树（如果 stat_prefix 非空，则为listener.<stat_prefix>. ），包含以下统计信息：

|Name                                            |Type              |Description|
|------------------------------------------------|----------------- |---------------------------------------------------------------------------------|
|downstream_cx_total                             |Counter           |连接总数|
|downstream_cx_destroy                           |Counter           |Total destroyed connections|
|downstream_cx_active                            |Gauge             |Total active connections|
|downstream_cx_length_ms                         |Histogram         |Connection length milliseconds|
|downstream_cx_transport_socket_connect_timeout  |Counter           |Total connections that timed out during transport socket connection negotiation|
|downstream_cx_overflow                          |Counter           |Total connections rejected due to enforcement of listener connection limit|
|downstream_cx_overload_reject                   |Counter           |Total connections rejected due to configured overload actions|
|downstream_global_cx_overflow                   |Counter           |Total connections rejected due to enforcement of global connection limit|
|connections_accepted_per_socket_event           |Histogram         |Number of connections accepted per listener socket event|
|downstream_pre_cx_timeout                       |Counter           |Sockets that timed out during listener filter processing|
|downstream_pre_cx_active                        |Gauge             |Sockets currently undergoing listener filter processing|
|extension_config_missing                        |Counter           |Total connections closed due to missing listener filter extension configuration|
|network_extension_config_missing                |Counter           |Total connections closed due to missing network filter extension configuration|
|global_cx_overflow                              |Counter           |Total connections rejected due to enforcement of the global connection limit|
|no_filter_chain_match                           |Counter           |Total connections that didn\'t match any filter chain|
|downstream_listener_filter_remote_close         |Counter           |Total connections closed by remote when peek data for listener filters|
|downstream_listener_filter_error                |Counter           |Total numbers of read errors when peeking data for listener filters|

# TLS statistics {#config_listener_stats_tls}

The following TLS statistics are rooted at *listener.\<address\>.ssl.*:

|Name                   |Type             |Description|
|-----------------------|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------|
|connection_error       |Counter          |Total TLS connection errors not including failed certificate verifications|
|handshake              |Counter          |Total successful TLS connection handshakes|
|session_reused         |Counter          |Total successful TLS session resumptions|
|no_certificate         |Counter          |Total successful TLS connections with no client certificate|
|fail_verify_no_cert    |Counter          |Total TLS connections that failed because of missing client certificate|
|fail_verify_error      |Counter          |Total TLS connections that failed CA verification|
|fail_verify_san        |Counter          |Total TLS connections that failed SAN verification|
|fail_verify_cert_hash  |Counter          |Total TLS connections that failed certificate pinning verification|
|ocsp_staple_failed     |Counter          |Total TLS connections that failed compliance with the OCSP policy|
|ocsp_staple_omitted    |Counter          |Total TLS connections that succeeded without stapling an OCSP response|
|ocsp_staple_responses  |Counter          |Total TLS connections where a valid OCSP response was available (irrespective of whether the client requested stapling)|
|ocsp_staple_requests   |Counter          |Total TLS connections where the client requested an OCSP staple|
|ciphers.\<cipher\>     |Counter          |Total successful TLS connections that used cipher \<cipher\>|
|curves.\<curve\>       |Counter          |Total successful TLS connections that used ECDHE curve \<curve\>|
|sigalgs.\<sigalg\>     |Counter          |Total successful TLS connections that used signature algorithm \<sigalg\>|
|versions.\<version\>   |Counter          |Total successful TLS connections that used protocol version \<version\>|
|was_key_usage_invalid  |Counter          |Total successful TLS connections that used an [invalid keyUsage extension]. (This is not avaiable in BoringSSL FIPS yet due to [issue #28246])|

# TCP statistics {#config_listener_stats_tcp}

The following TCP statistics, which are available when using the `TCP stats transport socket <envoy_v3_api_msg_extensions.transport_sockets.tcp_stats.v3.Config>`{.interpreted-text role="ref"},
are rooted at *listener.\<address\>.tcp_stats.*:

:::: note
::: title
Note
:::

These metrics are provided by the operating system. Due to differences in operating system metrics available and the methodology
used to take measurements, the values may not be consistent across different operating systems or versions of the same operating
system.
::::

|Name                                  |Type             |Description|
|--------------------------------------|-----------------|------------------------------------------------------------------------------------------------------|
|cx_tx_segments                        |Counter          |Total TCP segments transmitted|
|cx_rx_segments                        |Counter          |Total TCP segments received|
|cx_tx_data_segments                   |Counter          |Total TCP segments with a non-zero data length transmitted|
|cx_rx_data_segments                   |Counter          |Total TCP segments with a non-zero data length received|
|cx_tx_retransmitted_segments          |Counter          |Total TCP segments retransmitted|
|cx_rx_bytes_received                  |Counter          |Total payload bytes received for which TCP acknowledgments have been sent.|
|cx_tx_bytes_sent                      |Counter          |Total payload bytes transmitted (including retransmitted bytes).|
|cx_tx_unsent_bytes                    |Gauge            |Bytes which Envoy has sent to the operating system which have not yet been sent|
|cx_tx_unacked_segments                |Gauge            |Segments which have been transmitted that have not yet been acknowledged|
|cx_tx_percent_retransmitted_segments  |Histogram        |Percent of segments on a connection which were retransmistted|
|cx_rtt_us                             |Histogram        |Smoothed round trip time estimate in microseconds|
|cx_rtt_variance_us                    |Histogram        |Estimated variance in microseconds of the round trip time. Higher values indicated more variability.|

# UDP statistics {#config_listener_stats_udp}

The following UDP statistics are available for UDP listeners and are rooted at
*listener.\<address\>.udp.*:

|Name                            |Type             |Description|
|--------------------------------|-----------------|------------------------------------------------------------------|
|downstream_rx_datagram_dropped  |Counter          |Number of datagrams dropped due to kernel overflow or truncation|

# Per-handler Listener Stats {#config_listener_stats_per_handler}

Every listener additionally has a statistics tree rooted at *listener.\<address\>.\<handler\>.* which
contains *per-handler* statistics. As described in the
`threading model <arch_overview_threading>`{.interpreted-text role="ref"} documentation, Envoy has a threading model which
includes the *main thread* as well as a number of *worker threads* which are controlled by the
`--concurrency`{.interpreted-text role="option"} option. Along these lines, *\<handler\>* is equal to *main_thread*,
*worker_0*, *worker_1*, etc. These statistics can be used to look for per-handler/worker imbalance
on either accepted or active connections.

|Name                  |Type             |Description
|----------------------|-----------------|-------------------------------------------
|downstream_cx_total   |Counter          |Total connections on this handler.
|downstream_cx_active  |Gauge            |Total active connections on this handler.

# Listener manager {#config_listener_manager_stats}

The listener manager has a statistics tree rooted at *listener_manager.* with the following
statistics. Any `:` character in the stats name is replaced with `_`.

|Name                          |Type             |Description|
|------------------------------|-----------------|-------------------------------------------------------------------------------------------------------------|
|listener_added                |Counter          |Total listeners added (either via static config or LDS).|
|listener_modified             |Counter          |Total listeners modified (via LDS).|
|listener_removed              |Counter          |Total listeners removed (via LDS).|
|listener_stopped              |Counter          |Total listeners stopped.|
|listener_create_success       |Counter          |Total listener objects successfully added to workers.|
|listener_create_failure       |Counter          |Total failed listener object additions to workers.|
|listener_in_place_updated     |Counter          |Total listener objects created to execute filter chain update path.|
|total_filter_chains_draining  |Gauge            |Number of currently draining filter chains.|
|total_listeners_warming       |Gauge            |Number of currently warming listeners.|
|total_listeners_active        |Gauge            |Number of currently active listeners.|
|total_listeners_draining      |Gauge            |Number of currently draining listeners.|
|workers_started               |Gauge            |A boolean (1 if started and 0 otherwise) that indicates whether listeners have been initialized on workers.|

  [Listener]: #listener {#toc-listener}
  [TLS statistics]: #config_listener_stats_tls {#toc-config_listener_stats_tls}
  [TCP statistics]: #config_listener_stats_tcp {#toc-config_listener_stats_tcp}
  [UDP statistics]: #config_listener_stats_udp {#toc-config_listener_stats_udp}
  [Per-handler Listener Stats]: #config_listener_stats_per_handler {#toc-config_listener_stats_per_handler}
  [Listener manager]: #config_listener_manager_stats {#toc-config_listener_manager_stats}
  [invalid keyUsage extension]: https://github.com/google/boringssl/blob/6f13380d27835e70ec7caf807da7a1f239b10da6/ssl/internal.h#L3117
  [issue #28246]: https://github.com/envoyproxy/envoy/issues/28246

