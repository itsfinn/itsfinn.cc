---
title: "Envoy 介绍之(七)"
date: 2024-01-04T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---

原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(其他特性、其他协议)

- 其他特性: 本地限速、全局限速、带宽限速、脚本扩展、IP透传、压缩库

- 其他协议: gRPC、MongoDB、DynamoDB、Redis、Postgres)
<!--more-->

# 其他特性

## 本地限流

Envoy 支持对 L4 连接进行本地（非分布式）限流，通过
[本地限流监听器过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/listener_filters/local_rate_limit_filter#config-listener-filters-local-rate-limit)
和[本地限流网络过滤器实现](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/local_rate_limit_filter#config-network-filters-local-rate-limit)。
区别在于，*本地限流监听器过滤器<config_listener_filters_local_rate_limit>*在 TLS 握手和过滤器链匹配之前处理套接字。

Envoy 还支持通过 [HTTP 本地限流过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/local_rate_limit_filter#config-http-filters-local-rate-limit)
对 HTTP 请求进行本地限流。这可以在监听器级别或更具体的级别（例如：虚拟主机或路由级别）上激活。

最后，Envoy 还支持全局限流。本地限流可以与全局限流结合使用，以减少对全局限流服务的负载。

## 全局限流

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

### 每个连接或每个 HTTP 请求的限流

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

### 基于配额的限流

目前还没有开源的限流服务参考实现。限流配额扩展目前可用于 Google Cloud Rate Limit Service。

基于配额的全局限流只能应用于 HTTP 请求。Envoy 将使用 HTTP 过滤器
[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/rate_limit_quota_filter#config-http-filters-rate-limit-quota)
对请求进行分桶和请求配额分配。

限流配额服务[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/other_features/rate_limit#config-rate-limit-quota-service)。

## 带宽限速

Envoy 支持通过 HTTP 带宽限制过滤器对 HTTP 请求和响应进行本地（非分布式）带宽限制。
这可以在监听器级别或更具体的级别（例如：虚拟主机或路由级别）全局激活。

## 脚本

Envoy 支持 Lua 脚本作为专用 HTTP 过滤器的一部分。

## IP透传

### 什么是IP透传

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

### HTTP Headers

HTTP headers可以通过[x-forwarded-for](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-forwarded-for)
头携带请求的原始IP地址。
上游服务器可以使用此头来确定下游远程地址。
Envoy还可以使用此头选择由[Original Src HTTP Filter](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/other_features/ip_transparency#arch-overview-ip-transparency-original-src-http)
使用的IP地址。

HTTP头方法有一些缺点：

- 它仅适用于HTTP。

- 它可能不受上游主机的支持。

- 它需要仔细配置。

### 代理协议  
  
[HAProxy Proxy Protocol](http://www.haproxy.org/download/1.9/doc/proxy-protocol.txt)
定义了一种协议，用于在主TCP流之前通过TCP通信传输连接的元数据。
这些元数据包括源IP地址。
Envoy代理支持使用
[代理协议过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/listener_filters/proxy_protocol#config-listener-filters-proxy-protocol)
来利用这些信息，这有助于恢复下游远程地址，并将其传播到
[x-forwarded-for](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-forwarded-for)头部。
此外，它还可以与
[Original Src监听器过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/other_features/ip_transparency#arch-overview-ip-transparency-original-src-listener)
结合使用。最后，Envoy代理支持使用
[代理协议传输套接字](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/transport_sockets/proxy_protocol/v3/upstream_proxy_protocol.proto#extension-envoy-transport-sockets-upstream-proxy-protocol)
生成此头部信息。

这是一个设置套接字的示例配置：

```yaml
clusters:
- name: service1
  connect_timeout: 0.25s
  type: strict_dns
  lb_policy: round_robin
  transport_socket:
    name: envoy.transport_sockets.upstream_proxy_protocol
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.transport_sockets.proxy_protocol.v3.ProxyProtocolUpstreamTransport
      config:
        version: V1
      transport_socket:
        name: envoy.transport_sockets.raw_buffer
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.raw_buffer.v3.RawBuffer
  ...
```

如果你计划将此套接字与HTTP连接管理器结合使用，需要考虑一些因素。由于代理协议是基于连接的协议，因此下游客户端之间不会重用上游连接。连接到Envoy的每个客户端都会与上游服务器建立新的连接。在建立连接之初，下游客户端信息会在发送任何其他数据之前转发给上游（注：这包括在TLS握手发生之前）。如果可能的话，应优先使用x-forwarded-for标头，因为使用此方法，Envoy能够重用上游连接。由于Envoy对下游和上游连接的处理是分开的，因此最好对上游连接强制执行短暂的空闲超时，因为当下游连接关闭时，Envoy不会自动关闭相应的上游连接。

代理协议的一些缺点：

- 它只支持TCP协议。

- 它需要上游主机支持。

### 原始源监听器过滤器

在受控的部署中，使用原始源监听器过滤器可以将下游远程地址复制到上游连接上。没有将元数据添加到上游请求或流中。相反，上游连接本身将以下游远程地址作为其源地址建立。此过滤器将与任何上游协议或主机一起工作。然而，它需要相当复杂的配置，并且由于路由约束，可能不支持所有部署。

原始源过滤器的一些缺点：

- 它要求Envoy能够访问下游远程地址。
- 它的配置相对复杂。
- 由于连接池的限制，它可能会引入轻微的性能下降。
- 不支持Windows。

### 原始源HTTP过滤器

在受控的部署中，可以使用
[原始源HTTP过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/original_src_filter#config-http-filters-original-src)
将下游远程地址复制到上游连接上。此过滤器与
[原始源监听器过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/other_features/ip_transparency#arch-overview-ip-transparency-original-src-listener)
非常相似。主要区别在于，它可以从HTTP头中推断出原始源地址，这对于单个下游连接承载来自不同原始源地址的多个HTTP请求的情况非常重要。前端代理转发到边车代理的部署就是这种情况的示例。

此过滤器将与任何上游HTTP主机一起工作。然而，它需要相当复杂的配置，并且由于路由约束，可能不支持所有部署。

原始源过滤器的一些缺点：

- 它要求Envoy经过适当配置，以便从[x-forwarded-for](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-forwarded-for)标头中提取下游远程地址。
- 它的配置相对复杂。
- 由于连接池的限制，它可能会引入轻微的性能下降。

> **注意**
>
> 此功能不支持Windows。

## 压缩库

### 底层实现

目前，Envoy使用zlib、brotli和zstd作为压缩库。

> **注意**
>
> [zlib-ng](https://github.com/zlib-ng/zlib-ng)是一个分支，托管了几个包含新优化的第三方贡献。
> 这些优化被认为对[提高压缩性能](https://github.com/envoyproxy/envoy/issues/8448#issuecomment-667152013)很有用。
> 通过使用`--define zlib=ng` Bazel 选项，
> 可以将Envoy构建为使用[zlib-ng](https://github.com/zlib-ng/zlib-ng)而不是常规[zlib](http://zlib.net/)。
> 用于构建zlib-ng的相关构建选项可以在[这里](https://github.com/envoyproxy/envoy/blob/v1.28.0/bazel/foreign_cc/BUILD)评估。
> 目前，此选项仅在Linux上可用。

# 其他协议

## gRPC

gRPC

gRPC是由Google开发的RPC框架。它使用协议缓冲区作为底层序列化/IDL格式。在传输层，它使用HTTP/2或更高版本进行请求/响应复用。Envoy在传输层和应用程序层都为gRPC提供了第一流的支持：

- gRPC使用  trailer 来传递请求状态。Envoy是少数几个正确支持  trailer 的HTTP代理之一，因此也是少数几个能够传输gRPC请求和响应的代理之一。

- 对于某些语言，gRPC的运行时相对不成熟。请参阅[以下概述](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/other_protocols/grpc#arch-overview-grpc-bridging)，了解有助于将gRPC引入更多语言的过滤器。

- gRPC-Web由一个[过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/grpc_json_transcoder_filter#config-http-filters-grpc-json-transcoder)支持，该过滤器允许gRPC-Web客户端通过HTTP/1.1向Envoy发送请求，并将其代理到gRPC服务器。它正在积极开发中，预计将成为gRPC[桥接过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/grpc_http1_bridge_filter#config-http-filters-grpc-bridge)的继任者。

- gRPC-JSON转码器由一个[过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/grpc_web_filter#config-http-filters-grpc-web)支持，该过滤器允许RESTful JSON API客户端通过HTTP向Envoy发送请求，并将其代理到gRPC服务。

### gRPC桥接

Envoy支持多个gRPC桥接：

- [grpc_http1_bridge 过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/grpc_http1_bridge_filter#config-http-filters-grpc-bridge)
  允许gRPC请求通过HTTP/1.1发送到Envoy。Envoy然后将请求翻译为HTTP/2或HTTP/3，以便传输到目标服务器。响应被翻译回HTTP/1.1。安装后，桥接过滤器将收集每个RPC的统计数据以及全局HTTP统计数据的数组。

- [grpc_http1_reverse_bridge 过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/grpc_http1_reverse_bridge_filter#config-http-filters-grpc-http1-reverse-bridge)允许将gRPC请求发送到Envoy，并在发送到上游时将其转换为HTTP/1.1。响应在发送到下游时被转换回gRPC。此过滤器还可以选择性地管理gRPC帧头，使得上游无需了解gRPC。

- [connect_grpc_bridge 过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/connect_grpc_bridge_filter#config-http-filters-connect-grpc-bridge)允许将Buf Connect请求发送到Envoy。Envoy然后将请求翻译为gRPC以发送到上游。响应被转换回Buf Connect协议以发送回下游。需要时，HTTP/1.1请求将升级为HTTP/2或HTTP/3。

### gRPC服务

除了在数据平面使用gRPC外，Envoy还将其用于控制平面([从管理服务器获取配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/overview/overview#config-overview))以及过滤器(例如[限速](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/rate_limit_filter#config-http-filters-rate-limit)或授权检查)。我们将其称为gRPC服务。

在指定gRPC服务时，需要指定使用
[Envoy gRPC客户端](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/grpc_service.proto#envoy-v3-api-field-config-core-v3-grpcservice-envoy-grpc)
或[Google C++ gRPC客户端](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/grpc_service.proto#envoy-v3-api-field-config-core-v3-grpcservice-google-grpc)。
我们在下面讨论了这种选择的权衡。

Envoy gRPC客户端是gRPC的极简自定义实现，利用了Envoy的HTTP/2或HTTP/3上游连接管理。服务作为常规的Envoy[集群](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/cluster_manager#arch-overview-cluster-manager)进行指定，具有常规的
[超时处理、重试](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/http/http_connection_management#arch-overview-http-conn-man)、
端点[发现](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/operations/dynamic_configuration#arch-overview-dynamic-config-eds)/[负载均衡/故障转移](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/load_balancing/overview#arch-overview-load-balancing)/负载报告、
[断路器](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/circuit_breaking#arch-overview-circuit-break)、
[健康检查](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/health_checking#arch-overview-health-checking)、
[离群检测](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/outlier#arch-overview-outlier-detection)。
它们共享与Envoy数据平面相同的[连接池](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/connection_pooling#arch-overview-conn-pool)机制。同样，集群统计信息可用于gRPC服务。由于客户端是最小的，因此不包括高级gRPC功能，例如[OAuth2](https://oauth.net/2/)或[gRPC-LB](https://grpc.io/blog/loadbalancing)查找。

Google C++ gRPC客户端基于Google在[https://github.com/grpc/grpc](https://github.com/grpc/grpc)上提供的gRPC参考实现。它提供了在Envoy gRPC客户端中缺少的高级gRPC功能。Google C++ gRPC客户端执行自己的负载均衡、重试、超时、端点管理等，与Envoy的集群管理无关。Google C++ gRPC客户端还支持自定义身份验证插件。

建议在大多数情况下使用Envoy gRPC客户端，而不需要Google C++ gRPC客户端中的高级功能。这提供了配置和监控的简单性。如果Envoy gRPC客户端缺少必要的功能，则应使用Google C++ gRPC客户端。

## MongoDB

Envoy支持一个网络级别的MongoDB嗅探过滤器，具有以下功能：

- MongoDB有线格式BSON解析器。

详细的MongoDB查询/操作统计信息，包括时间戳和路由集群的散列/多获取计数(including timings and scatter/multi-get counts for routed clusters.)。

- 查询日志记录。

- 通过$comment查询参数的每个调用方的统计信息。

- 故障注入。

MongoDB过滤器是Envoy可扩展性和核心抽象的一个很好的例子，可用于在所有应用程序和MongoDB数据库之间进行过滤。它提供了一个宝贵的数据源，与应用程序平台和使用的特定MongoDB驱动程序无关。

MongoDB 代理过滤器[配置指南](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/mongo_proxy_filter#config-network-filters-mongo-proxy)

## DynamoDB

Envoy支持一个HTTP级别的 DynamoDB 嗅探过滤器，具有以下功能：

- DynamoDB API请求/响应解析器。

- DynamoDB每个操作/每个表/每个分区的统计信息以及操作统计信息。

- 从响应JSON解析的4xx响应的失败类型统计信息，例如ProvisionedThroughputExceededException。

- 批处理操作的部分失败统计信息。

DynamoDB过滤器是Envoy在HTTP层上的可扩展性和核心抽象的一个很好的例子，可用于过滤所有应用程序与DynamoDB的通信。它提供了一个宝贵的数据源，与应用程序平台和使用的特定AWS SDK无关。

DynamoDB 过滤器[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/dynamodb_filter#config-http-filters-dynamo)

## Redis

Envoy可以作为Redis代理，在集群的实例之间划分命令。在这种模式下，Envoy的目标是在可用性和分区容忍度之间保持一致性。这是将Envoy与Redis Cluster进行比较的关键点。Envoy被设计为最佳努力缓存，这意味着它不会尝试协调不一致的数据或保持集群成员的全局一致性视图。它还支持根据不同工作负载的访问模式、驱逐或隔离要求，将命令从不同的工作负载路由到不同的上游集群。

Redis项目提供了关于分区与Redis相关的全面参考。请参阅“[分区：如何将数据拆分到多个Redis实例中](https://redis.io/topics/partitioning)”。

以下是Envoy Redis的一些功能：

- [Redis协议](https://redis.io/topics/protocol)编解码器。
- 基于哈希的分区。
- Redis事务支持。
- Ketama分布。
- 详细的命令统计信息。
- 主动和被动健康检查。
- 哈希标签。
- 前缀路由。
- 独立的下游客户端和上游服务器身份验证。
- 所有请求或仅写请求的请求镜像。
- 控制[读取请求的路由](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/redis_proxy/v3/redis_proxy.proto#envoy-v3-api-field-extensions-filters-network-redis-proxy-v3-redisproxy-connpoolsettings-read-policy)。仅与Redis Cluster一起工作。

计划中的未来增强功能：

- 其他时间统计信息。
- 断路器。
- 对碎片化命令进行请求折叠。
- 复制。
- 内置重试。
- 跟踪。

### 配置

有关过滤器配置的详细信息，请参阅Redis代理过滤器[配置参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/redis_proxy_filter#config-network-filters-redis-proxy)。

相应的集群定义应配置为[环哈希负载均衡](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-lb-policy)。

如果需要[主动健康检查](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/health_checking#arch-overview-health-checking)，
集群应配置为
[自定义健康检查](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/health_check.proto#envoy-v3-api-field-config-core-v3-healthcheck-custom-health-check)，
并将其配置为
[Redis健康检查器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/health_checkers/redis#config-health-checkers-redis)。

如果需要被动健康检查，还应配置[异常检测](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/outlier#arch-overview-outlier-detection)。

对于被动健康检查，连接超时、命令超时和连接关闭映射为5xx。来自Redis的所有其他响应都被计为成功。

### Redis集群支持

Envoy提供了对Redis集群的支持。

当使用Envoy作为Redis集群的边车代理时，服务可以使用任何语言的非集群Redis客户端连接到代理，就像它是一个单节点Redis实例一样。Envoy代理将跟踪集群拓扑并根据规范将命令发送到正确的Redis节点。还可以向Envoy代理添加高级功能，例如从副本读取，而不是在每种语言中更新Redis客户端。

Envoy代理通过向集群中的随机节点发送定期的集群插槽命令来跟踪集群拓扑，并维护以下信息：

- 已知节点的列表。

- 每个分片的初级节点。

- 进入或离开集群的节点。

Envoy代理支持通过`cluster slots`命令响应中的IP地址和主机名标识节点。在无法解析主机名的情况下，Envoy将定期重试解析所有节点，直到成功。无法解析副本只会跳过该副本。另一方面，如果设置了[enable_redirection](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/redis_proxy/v3/redis_proxy.proto#envoy-v3-api-field-extensions-filters-network-redis-proxy-v3-redisproxy-connpoolsettings-enable-redirection)，并且收到包含主机名的MOVED或ASK响应，Envoy不会自动进行DNS查找，而是将错误原样传递给客户端。要使Envoy进行DNS查找并遵循重定向，您需要在连接池设置中配置DNS缓存选项[dns_cache_config](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/redis_proxy/v3/redis_proxy.proto#envoy-v3-api-field-extensions-filters-network-redis-proxy-v3-redisproxy-connpoolsettings-dns-cache-config)。有关如何启用重定向的DNS查找的配置示例，请参阅过滤器[配置参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/redis_proxy_filter#config-network-filters-redis-proxy)。

有关拓扑配置的详细信息，请参阅Redis Cluster [v3 API参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/clusters/redis/v3/redis_cluster.proto#envoy-v3-api-msg-extensions-clusters-redis-v3-redisclusterconfig)。

每个Redis集群都有自己的额外统计树，其根在 cluster.<name>.redis_cluster. 下，具有以下统计信息：

| 名称                                     | 类型      | 描述                                                                                                                                                                       |
| ---------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| max_upstream_unknown_connections_reached | Counter   | 在达到连接池的max_upstream_unknown_connections限制后，无法创建到未知主机的上游连接的次数。经过重定向后，不创建上游到未知主机的连接。                              |
| upstream_cx_drained                      | Counter   | 在关闭之前，排空所有活跃请求的上游连接的总数。排空所有活跃请求的上游连接。                                                                          |
| upstream_commands.upstream_rq_time       | Histogram   | 所有类型请求的上游请求时间的直方图。所有类型请求的上游请求时间直方图。                                                                          |

通过设置 [enable_command_stats](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/redis_proxy/v3/redis_proxy.proto#envoy-v3-api-field-extensions-filters-network-redis-proxy-v3-redisproxy-connpoolsettings-enable-command-stats) 开启每个集群的命令统计：

| 名称                                | 类型      | 描述                                                                  |
| ----------------------------------- | --------- | --------------------------------------------------------------------- |
| upstream_commands.[command].success | Counter   | 特定 Redis 命令成功请求的总数                   |
| upstream_commands.[command].failure | Counter   | 特定 Redis 命令失败或取消的请求总数          |
| upstream_commands.[command].total   | Counter   | 特定 Redis 命令的请求总数（成功和失败的总和） |
| upstream_commands.[command].latency | Histogram | 特定 Redis 命令请求的延迟 |

### 事务

Envoy 支持事务 (MULTI)。其使用方式与常规 Redis 相同：您使用 MULTI 开始事务，然后使用 EXEC 执行它。在事务中，只支持 Envoy 支持的命令（见下文）和单键命令，即不支持 MGET 和 MSET。DISCARD 命令也支持。

当在 Redis 集群模式下工作时，Envoy 会将事务中的所有命令转发给处理事务中的第一个基于键的命令的节点。用户有责任确保事务中的所有键都映射到相同的哈希槽，因为命令不会被重定向。

### 支持的命令

在协议层面，支持管道。在可能的情况下使用管道以获得最佳性能。

在命令级别，Envoy 只支持能够可靠地哈希到服务器的命令。AUTH 和 PING 是唯一的例外。如果下游配置了密码，则 AUTH 将由 Envoy 在本地处理，并且在身份验证成功之前，不会处理其他命令。如果为集群配置了上游服务器身份验证密码，Envoy 将与上游服务器连接时透明地发出 AUTH 命令。Envoy 会立即对 PING 做出响应，发送 PONG。不允许 PING 的参数。所有其他受支持的命令必须包含一个键。受支持的命令在功能上与原始 Redis 命令相同，除非在故障场景中可能有所不同。

每个命令的使用参考官方 [Redis 命令参考](https://redis.io/commands)

[支持的命令](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/other_protocols/redis#supported-commands)列表

### 故障模式

如果 Redis 抛出错误，我们将该错误作为命令的响应传递。Envoy 将 Redis 带有错误数据类型的响应视为正常响应，并将其传递给调用者。

Envoy 还可以在响应客户端时生成自己的错误。

Envoy 错误对照表

| 错误 | 含义 |
| --- | ---|
| no upstream host | 在为该键选择的环位置上，环哈希负载均衡器没有可用的健康主机。 |
| upstream failure | 后端未在超时时间内响应或关闭了连接。 |
| invalid request | 由于数据类型或长度，命令被命令拆分器的第一阶段拒绝。 |
| unsupported command | 该命令未被 Envoy 识别，因此无法提供服务，因为它无法被哈希到后端服务器。 |
| finished with n errors | 分片命令（例如 DEL）将返回收到的错误总数，如果有收到任何错误。 |
| upstream protocol error | 分片命令收到了意外的数据类型或后端响应不符合 Redis 协议。 |
| wrong number of arguments for command | 某些命令在 Envoy 中检查参数的数量是否正确。 |
| NOAUTH Authentication required. | 该命令被拒绝，因为设置了下游认证密码，而客户端未成功通过身份验证。 |
| ERR invalid password | 由于密码无效，认证命令失败。 |
| ERR Client sent AUTH, but no password is set | 收到了认证命令，但没有配置下游认证密码。 |

在 MGET 的情况下，无法获取的每个单独的键都会生成一个错误响应。例如，如果我们获取五个键，其中两个键的后端超时，我们将在每个值的位置上获得一个错误响应。

```shell
$ redis-cli MGET a b c d e
1) "alpha"
2) "bravo"
3) (error) upstream failure
4) (error) upstream failure
5) "echo"
```

## Postgres

Envoy 支持一个网络级别的 Postgres 嗅探过滤器，以增加网络可观察性。通过使用 Postgres 代理，Envoy 能够解码 Postgres 前端/后端协议，并从解码的信息中收集统计数据。

Postgres 过滤器的主要目标是捕获运行时统计信息，而不会对 Postgres 上游服务器产生任何影响或生成任何负载，它对上游服务器是透明的。该过滤器目前提供以下功能：

- 解码非 SSL 流量，忽略 SSL 流量。
- 解码会话信息。
- 在转发上游之前对传入非 SSL 流量进行编码。
- 捕获事务信息，包括提交和回滚。
- 公开不同类型的语句（INSERT、DELETE、UPDATE 等）的计数器。这些计数器根据解码的后端 CommandComplete 消息进行更新，而不是通过解码客户端发送的 SQL 语句。
- 计算前端、后端和未知消息的数量。
- 识别错误和通知后端响应。

Postgres 过滤器解决了 Postgres 部署中的一个显著问题：收集这些信息要么给服务器带来额外的负载；要么需要从服务器拉取查询元数据的查询，有时需要外部组件或扩展。此过滤器提供宝贵的可观察性信息，而不会影响上游 Postgres 服务器的性能或要求安装任何软件。

Postgres 代理过滤器[配置参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/postgres_proxy_filter#config-network-filters-postgres-proxy)。
