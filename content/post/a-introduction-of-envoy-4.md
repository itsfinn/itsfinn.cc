---
title: "Envoy 介绍之(四)"
date: 2023-12-27T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---

原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(可观测性)
<!--more-->

# 统计数据

Envoy的核心目标之一是提升网络的可观察性。根据其配置的不同，Envoy能生成一系列的统计数据。这些数据大致分为三个类别：

- **下游**：涉及到入站连接和请求的统计，由监听器、HTTP连接管理器、TCP代理过滤器等组件生成。

- **上游**：涉及到出站连接和请求的统计，由连接池、路由过滤器、TCP代理过滤器等组件生成。

- **服务器**：反映Envoy服务器实例运行状态的统计，如服务器运行时间、内存分配情况等都在此类别中。

在典型的简单代理使用场景中，会同时涉及下游和上游的统计数据。通过这两种类型的数据，我们可以详尽地了解网络中的每一跳。
网格整体的统计数据为我们提供了每一跳以及整个网络健康状况的详细视图。这些统计数据在操作手册中有详细的文档说明。

自v2 API版本起，Envoy开始支持自定义的、可插入的统计数据收集器。Envoy已经内置了[一些标准的统计收集器实现](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/metrics/v3/stats.proto#envoy-v3-api-msg-config-metrics-v3-statssink)。
其中一些收集器还能够支持输出带有标签或维度的统计信息。

在Envoy的内部及其文档中，统计信息都通过一个标准的字符串格式来识别。
这些字符串中的动态部分会被转换为标签。用户可以通过[Tag Specifier 配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/metrics/v3/stats.proto#envoy-v3-api-msg-config-metrics-v3-tagspecifier)来自定义这些标签的生成规则。

Envoy会生成如下三种类型的统计数据：

- **计数器**：只会增加不会减少的无符号整数值，如总请求数。

- **仪表**：可以增加也可以减少的无符号整数值，如当前活跃请求数。

- **直方图**：一个无符号整数数据流的多个切分，由数据收集器进行聚合处理，最终得到汇总的百分位数值，如上游请求处理时间。

内部实现上，为了提升性能，计数器和仪表会被批量处理并定期发送到统计数据库。直方图数据则在产生时即刻记录。
需要注意的是，过去被称为计时器的统计项现在已经并入直方图，因为它们的唯一区别在于度量单位的不同。

- [v3 API参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-field-config-bootstrap-v3-bootstrap-stats-sinks)。

# 访问日志

[HTTP连接管理器](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/http/http_connection_management#arch-overview-http-conn-man)、
[TCP代理](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/listeners/listener_filters#arch-overview-tcp-proxy)
和[Thrift代理](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/thrift_proxy_filter#config-network-filters-thrift-proxy)
都支持带有以下特点的可扩展访问日志记录功能：

- 对于每个连接流，可以配置多条访问日志。

- 通过自定义访问日志过滤器，可以根据请求和响应的不同类型，选择性地记录到不同的访问日志中。

使用监听器访问日志可以实现对下游连接的日志记录。这种监听器日志是对HTTP请求日志的补充，并且可以与过滤器访问日志独立开启。

如果开启了访问日志，那么在TCP连接结束或HTTP流结束时，日志默认会发送到配置好的日志系统中。
此外，还可以通过扩展功能，使得访问日志能够在TCP连接建立或HTTP流开始时就开始记录，或者定期发送日志。
在建立上游连接后或接收到新的HTTP请求后立即记录日志的功能，不需要依赖于定期发送日志的设置，两者可以独立操作。

## 会话开始访问日志记录

### TCP代理

在TCP代理中，可以设置在与上游服务器成功建立连接之后立刻记录一条访问日志，这通过启用
[建立连接时立即记录访问日志](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/tcp_proxy/v3/tcp_proxy.proto#envoy-v3-api-field-extensions-filters-network-tcp-proxy-v3-tcpproxy-tcpaccesslogoptions-flush-access-log-on-connected)
的选项来实现。

### HTTP连接管理器

对于HTTP连接管理器，可以在收到新的HTTP请求并且在开始处理过滤器链之前记录一条访问日志，通过启用
[接收新请求时记录访问日志](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-hcmaccesslogoptions-flush-access-log-on-new-request)
的选项来实现。需要注意的是，此时还无法获取到某些信息，例如上游服务器的信息。

### HTTP路由过滤器

在HTTP路由过滤器中，可以在下游流与新的上游流关联且与上游服务器成功建立连接后记录上游访问日志，这通过启用
[上游流建立时记录日志](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/http/router/v3/router.proto#envoy-v3-api-field-extensions-filters-http-router-v3-router-upstreamaccesslogoptions-flush-upstream-log-on-upstream-stream)
的选项来实现。请注意，如果HTTP请求中有重试行为，那么每一次重试的开始都会记录一条新的上游访问日志。

## 周期性访问日志

### TCP代理

TCP代理支持配置周期性访问日志，通过设置访问日志的记录间隔实现。
注意：首条日志将在接收新连接一个间隔后记录，不论是否已经建立了上游连接。

### HTTP连接管理器

HTTP连接管理器也支持配置周期性访问日志，同样是通过设置记录间隔来控制。
注意：首条日志将在接收新的HTTP请求一个间隔后记录，发生在处理HTTP过滤器链之前，不论是否建立了上游连接。

### HTTP路由过滤器

路由过滤器同样可以启用周期性访问日志，通过配置上游日志的记录间隔来实现。
注意：首条日志将在路由过滤器接收到新的HTTP请求一个间隔后记录，不论是否建立了上游连接。

## 访问日志过滤器

Envoy提供了多种内置的访问日志过滤器，并支持运行时注册扩展过滤器。

## 访问日志输出方式

Envoy支持多种可扩展的访问日志输出方式。当前支持的输出方式包括：

### 文件

基于异步IO的刷新机制，确保访问日志的记录不会阻塞主要的网络处理线程。

允许使用预定义的字段和任意HTTP请求与响应的头部信息来自定义访问日志的格式。

### gRPC

Envoy能够把访问日志信息发送至gRPC的访问日志服务。

### 标准输出

基于异步IO的刷新机制，确保访问日志的记录不会阻塞主要的网络处理线程。

允许使用预定义的字段和任意HTTP请求与响应的头部信息来自定义访问日志的格式。

日志将被写入进程的标准输出流，适用于所有平台。

### 标准错误

基于异步IO的刷新机制，确保访问日志的记录不会阻塞主要的网络处理线程。

允许使用预定义的字段和任意HTTP请求与响应的头部信息来自定义访问日志的格式。

日志将被写入进程的标准错误流，适用于所有平台。

## 进一步阅读

- 访问日志[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/observability/access_log/usage#config-access-log)
- 文件[访问日志输出](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/access_loggers/file/v3/file.proto#envoy-v3-api-msg-extensions-access-loggers-file-v3-fileaccesslog)
- gRPC[访问日志服务(ALS)](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/access_loggers/grpc/v3/als.proto#envoy-v3-api-msg-extensions-access-loggers-grpc-v3-httpgrpcaccesslogconfig)输出方式的文档。
- OpenTelemetry（gRPC）[LogsService](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/access_loggers/open_telemetry/v3/logs_service.proto#envoy-v3-api-msg-extensions-access-loggers-open-telemetry-v3-opentelemetryaccesslogconfig)的文档。
- 标准输出[访问日志输出](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/access_loggers/stream/v3/stream.proto#envoy-v3-api-msg-extensions-access-loggers-stream-v3-stdoutaccesslog)。
- 标准错误[访问日志输出](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/access_loggers/stream/v3/stream.proto#envoy-v3-api-msg-extensions-access-loggers-stream-v3-stderraccesslog)。
