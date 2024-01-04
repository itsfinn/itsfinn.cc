---
title: "Envoy 介绍之(七)"
date: 2023-01-04T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---

原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(其他特性): 本地限速、全局限速、带宽限速、脚本扩展、IP透传、压缩库
<!--more-->

# 本地限流

Envoy 支持对 L4 连接进行本地（非分布式）限流，通过
[本地限流监听器过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/listener_filters/local_rate_limit_filter#config-listener-filters-local-rate-limit)
和[本地限流网络过滤器实现](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/local_rate_limit_filter#config-network-filters-local-rate-limit)。
区别在于，*本地限流监听器过滤器<config_listener_filters_local_rate_limit>*在 TLS 握手和过滤器链匹配之前处理套接字。

Envoy 还支持通过 [HTTP 本地限流过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/local_rate_limit_filter#config-http-filters-local-rate-limit)
对 HTTP 请求进行本地限流。这可以在监听器级别或更具体的级别（例如：虚拟主机或路由级别）上激活。

最后，Envoy 还支持全局限流。本地限流可以与全局限流结合使用，以减少对全局限流服务的负载。

# 全局限流

虽然分布式[断路器](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/circuit_breaking#arch-overview-circuit-break)
在控制分布式系统中的吞吐量方面通常非常有效，
但在某些情况下它并不是很有效，这时就需要全局限流。
最常见的情况是大量主机转发到少数主机，平均请求延迟很低（例如，与数据库服务器的连接/请求）。
如果目标主机变得拥堵，下游主机将淹没上游集群。
在这种情况下，很难在每个下游主机上配置足够严格的断路器限制，
以确保系统在典型请求模式下的正常运行，同时在系统开始出现故障时仍能防止级联故障。
全局限流是解决这种情况的好方案。

Envoy 提供了两种全局限流实现：

1. 每个连接或每个 HTTP 请求的限流检查。

2. 基于配额的，带有定期负载报告，允许在多个 Envoy 实例之间公平地共享全局限流。
   这种实现适用于大型 Envoy 部署，其中每秒请求负载很高，并且可能无法在所有 Envoy 实例之间均匀平衡。

## 每个连接或每个 HTTP 请求的限流

Envoy 直接与全局 gRPC 限流服务集成。
虽然任何实现定义的 RPC/IDL 协议的服务都可以使用，
但 Envoy 提供了一个用 Go 编写的[参考实现](https://github.com/envoyproxy/ratelimit)，
该实现使用 Redis 后端。Envoy 的限流集成具有以下功能：

- **网络级别限流过滤器**：对于安装了过滤器的监听器上的每个新连接，Envoy 将调用限流服务。
  配置指定了要限速的特定域和描述符集。这最终将对通过监听器的每秒连接数进行限速。
  [配置参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/rate_limit_filter#config-network-filters-rate-limit)。

- **HTTP 级别限流过滤器**：对于安装了过滤器的监听器上的每个新请求，Envoy 将调用限流服务，
  并且路由表指定应该调用全局限流服务。目标上游集群的所有请求以及从原始集群到目标集群的所有请求都可以进行限速。
  [配置参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/rate_limit_filter#config-http-filters-rate-limit)

限流服务[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/other_features/rate_limit#config-rate-limit-service)。

请注意，Envoy 还支持[本地限流](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/local_rate_limit_filter#config-network-filters-local-rate-limit)。
本地限流可以与全局限流结合使用，以减少对全局限流服务的负载。
例如，本地令牌桶限流可以吸收可能淹没全局限流服务的非常大的负载突发。
因此，限流分为两个阶段进行。初始粗粒度限制由令牌桶限制执行，然后由细粒度全局限制完成工作。

## 基于配额的限流

目前还没有开源的限流服务参考实现。限流配额扩展目前可用于 Google Cloud Rate Limit Service。

基于配额的全局限流只能应用于 HTTP 请求。Envoy 将使用 HTTP 过滤器
[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/rate_limit_quota_filter#config-http-filters-rate-limit-quota)
对请求进行分桶和请求配额分配。

限流配额服务[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/other_features/rate_limit#config-rate-limit-quota-service)。

# 带宽限速

Envoy 支持通过 HTTP 带宽限制过滤器对 HTTP 请求和响应进行本地（非分布式）带宽限制。
这可以在监听器级别或更具体的级别（例如：虚拟主机或路由级别）全局激活。

# 脚本

Envoy 支持 Lua 脚本作为专用 HTTP 过滤器的一部分。

# IP透传

## 什么是IP透传

作为一个代理，Envoy是一个IP端点：它有自己的IP地址，与任何下游请求的IP地址不同。
因此，当Envoy建立与上游主机的连接时，该连接的IP地址将与任何代理连接的IP地址不同。

有时，上游服务器或网络出于多种原因，可能需要知道原始的IP地址，称为*下游远程地址*，。一些例子包括：

- 原始IP地址作为身份标识的一部分，

- 原始IP地址用于实施网络策略，或

- 原始IP地址用于审计。

Envoy支持多种方法将下游远程地址提供给上游主机。这些技术的复杂性和适用性各不相同。

Envoy还支持用于检测原始IP地址的[扩展](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-original-ip-detection-extensions)。
这可能在以下情况中非常有用：下面列出的任何技术都不适用于您的设置。
可用的扩展有两个：
[自定义头扩展](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/http/original_ip_detection/custom_header/v3/custom_header.proto#envoy-v3-api-msg-extensions-http-original-ip-detection-custom-header-v3-customheaderconfig)
和[xff扩展](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/http/original_ip_detection/xff/v3/xff.proto#envoy-v3-api-msg-extensions-http-original-ip-detection-xff-v3-xffconfig)。

## HTTP Headers

HTTP headers可以通过[x-forwarded-for](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-forwarded-for)
头携带请求的原始IP地址。
上游服务器可以使用此头来确定下游远程地址。
Envoy还可以使用此头选择由[Original Src HTTP Filter](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/other_features/ip_transparency#arch-overview-ip-transparency-original-src-http)
使用的IP地址。

HTTP头方法有一些缺点：

- 它仅适用于HTTP。

- 它可能不受上游主机的支持。

- 它需要仔细配置。

## 代理协议  
  
[HAProxy Proxy Protocol](http://www.haproxy.org/download/1.9/doc/proxy-protocol.txt)
定义了一种协议，用于在主TCP流之前通过TCP通信传输连接的元数据。
这些元数据包括源IP地址。
Envoy代理支持使用代理协议过滤器来利用这些信息，这有助于恢复下游远程地址，并将其传播到x-forwarded-for头部。
此外，它还可以与Original Src监听器过滤器结合使用。最后，Envoy代理支持使用代理协议传输套接字生成此头部信息。
