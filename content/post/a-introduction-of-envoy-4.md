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

# 追踪(Tracing)

## 概述

分布式追踪技术允许开发者在庞大的面向服务的系统中追踪并可视化调用过程，这对于分析数据序列化、任务并行处理以及延迟问题的根源至关重要。Envoy提供了三项与系统范围内追踪相关的功能：

- 请求ID生成：Envoy会根据需要生成通用唯一识别码（UUID），并将其添加到HTTP头部
  [x-request-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-request-id)中。
  应用可以转发此头部信息，以实现日志的统一管理和追踪。这一功能可以通过扩展在每个
  [HTTP连接管理器](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-request-id-extension)
  中单独配置。

- 客户端追踪ID串联：HTTP头部
  [x-client-trace-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-client-trace-id)
  可以用于将外部的不受信任请求ID与内部的受信任的
  [x-request-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-request-id)
  进行串联。

- 外部追踪服务整合：Envoy支持接入多种可插拔的外部追踪可视化工具，主要分为两类：

  * 直接包含在Envoy代码库中的外部追踪工具，例如Zipkin、Jaeger、Datadog、SkyWalking和AWS X-Ray。

  * 作为第三方插件提供的外部追踪工具，例如Instana。

## 如何启动追踪

要启动追踪，管理请求的HTTP连接管理器需要配置追踪功能。
追踪可以通过以下几种方式启动：

- 外部客户端通过
  [x-client-trace-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-client-trace-id)头部传入。
- 内部服务通过
  [x-envoy-force-trace](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-envoy-force-trace)
  头部传入。
- 通过[random_sampling](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/runtime#config-http-conn-man-runtime-random-sampling)配置进行随机采样启动。

## 追踪上下文的传递

Envoy能够记录服务网格内部通信的追踪信息。为了将调用链中
各个代理产生的追踪信息串联起来，服务需要在处理入站和
出站请求时传递追踪上下文。

不论采用何种追踪系统，服务都应该转发
[x-request-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-request-id)
头部，以便能够跨服务关联日志信息。

> **注意**
> 
> Envoy的请求ID生成机制可通过扩展进行定制，默认采用[UuidRequestIdConfig](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/request_id/uuid/v3/uuid.proto#envoy-v3-api-msg-extensions-request-id-uuid-v3-uuidrequestidconfig)实现。
> 这一扩展的配置需在[HTTP连接管理器](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-request-id-extension)的相关配置项中设置（具体例子请参考文档）。
> 默认实现会修改UUID4格式的请求ID，以包含追踪原因。此功能确保了在Envoy集群中
> 进行一致的采样，正如[x-request-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-request-id)头部文档所描述。然而，如果要保持外部生成的
> 请求ID，则这种追踪原因的打包可能会导致问题。可以通过设置[pack_trace_reason](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/request_id/uuid/v3/uuid.proto#envoy-v3-api-field-extensions-request-id-uuid-v3-uuidrequestidconfig-pack-trace-reason)
> 为false来关闭这一行为，但这样也会关闭部署中的稳定追踪原因传播及其相关特性。

> **注意**
> 
> Envoy的采样策略通常是根据[x-request-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-request-id)的值来设定的。但这种策略仅适用于
> 全部由Envoy组成的集群。如果集群中包含非Envoy的服务代理，那么采样将不会
> 考虑这些代理的策略。在由多种服务代理组成的网格中，更有效的方法是不使用
> Envoy的采样策略，而是根据追踪提供者的策略进行采样。这可以通过将
> [use_request_id_for_trace_sampling](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/request_id/uuid/v3/uuid.proto#envoy-v3-api-field-extensions-request-id-uuid-v3-uuidrequestidconfig-use-request-id-for-trace-sampling)设置为false来实现。

为了在追踪过程中确立 span（代表工作的逻辑单元）之间的父子层级关系，
追踪系统需要携带额外的上下文信息。服务可以直接集成LightStep（通过
OpenTelemetry API）或Zipkin追踪器，从接收到的请求中提取追踪上下文，
并将其嵌入到随后发出的请求中。这种做法还允许服务记录内部工作的额外跨度，
这对于全链路追踪的分析非常有帮助。

服务还可以选择手动传递追踪上下文：

- 当使用LightStep追踪器时，Envoy依靠服务在HTTP请求中传递
  [x-ot-span-context](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-ot-span-context)头部，以实现对其他服务的追踪。
- 当使用Zipkin追踪器时，Envoy依靠服务传递B3系列的HTTP头部
  ([x-b3-traceid](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-b3-traceid)、
  [x-b3-spanid](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-b3-spanid)、
  [x-b3-parentspanid](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-b3-parentspanid)、
  [x-b3-sampled](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-b3-sampled)
  和[x-b3-flags](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-b3-flags))。
  外部客户端也可以通过
  [x-b3-sampled](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-b3-sampled)
  头部来控制请求的追踪
  启用或禁用。同时，Zipkin还支持更简洁的单一
  [b3](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-b3)
  头部传播格式。
- 当使用Datadog追踪器时，Envoy依靠服务传递Datadog特有的HTTP头部
  ([x-datadog-trace-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-datadog-trace-id)、
  [x-datadog-parent-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-datadog-parent-id)、
  [x-datadog-sampling-priority](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-datadog-sampling-priority))。
- 当使用SkyWalking追踪器时，Envoy依靠服务传递SkyWalking特有的HTTP头部
  ([sw8](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-sw8))。
- 当使用AWS X-Ray追踪器时，Envoy依靠服务传递X-Ray特有的HTTP头部
  ([x-amzn-trace-id](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-amzn-trace-id))。

## 追踪中的数据内容

端到端追踪包含了多个跨度(span)。每个跨度(span)是一项具有明确开始时间和持续时间的
逻辑工作单元，并且可能包含相关的元数据。Envoy生成的每个跨度(span)都会包括：

- 由`--service-cluster`参数指定的服务集群源。
- 请求的开始时间和持续时长。
- 由`--service-node`参数指定的起始主机。
- 通过[x-envoy-downstream-service-cluster](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-downstream-service-cluster)头部指定的下游服务集群。
- HTTP请求的URL、方法、协议和用户代理信息。
- 通过[custom_tags](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-tracing-custom-tags)添加的额外自定义标签。
- 上游集群的名称、可观测性名称和地址。
- HTTP响应的状态码。
- 如果可用，GRPC响应的状态和消息。
- 当HTTP状态码为5xx或GRPC状态表示服务端错误时，会添加一个错误标签。
  有关GRPC状态码的更多信息，请参阅[GRPC文档](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)。
- 特定于追踪系统的元数据。

每个跨度还包括一个默认定义为被调用服务主机名的名称（或操作名），
但可以通过路由的[config.route.v3.Decorator](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-decorator)进行定制。名称也可以
通过[x-envoy-decorator-operation](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/router_filter#config-http-filters-router-x-envoy-decorator-operation)头部覆盖。

Envoy会自动把跨度发送到追踪收集器。根据不同的追踪收集器，
多个跨度会根据[x-request-id]((https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/headers#config-http-conn-man-headers-x-request-id))(LightStep)或追踪ID配置
（Zipkin和Datadog）等共同信息进行关联。要了解如何在Envoy中配置追踪，
请参考[v3 API相关文档](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/trace/v3/http_tracer.proto#envoy-v3-api-msg-config-trace-v3-tracing)。

## Baggage

Baggage是一种机制，它允许数据在整个追踪过程中一直可用。尽管标签等元数据
一般是通过非直接方式传递给收集器，但Baggage数据则被直接注入到请求的上下文中，
并在请求过程中对应用程序可访问。这意味着元数据可以不依赖于特定应用的修改，
就从请求的开始自然地穿越整个服务网格。有关Baggage的更多信息，请参阅
[OpenTelemetry的文档](https://opentelemetry.io/docs/concepts/signals/baggage/)。

不同的追踪提供者对于处理Baggage的能力各不相同：

- Lightstep（以及任何符合OpenTelemetry标准的追踪器）可以读取和写入Baggage。
- Zipkin目前还不支持Baggage。
- X-Ray和OpenCensus不支持Baggage。

## 不同类型的Span

正如前文所提到的，一个追踪是由多个Span组成的，这些Span可能属于不同类型。
SkyWalking、Zipkin和OpenTelemetry等追踪系统提供了相似的Span类型。
最常见的类型是CLIENT和SERVER。CLIENT类型的Span由客户端生成，对应于发送到
服务器的请求；而SERVER类型的Span则由服务器生成，对应于从客户端接收的请求。

一个基本的追踪链条如下所示。通常情况下，一个SERVER类型Span的父Span应该是
CLIENT类型。链条中的每一环都必须确保Span类型的准确性。

```
-> [SERVER -> CLIENT] -> [SERVER -> CLIENT] -> ...
          App A                 App B
```

## Envoy的应用模式

Envoy在服务网格中经常以sidecar形式出现，了解Envoy的不同追踪模式至关重要。

在第一种模式下，Envoy作为sidecar部署。这时，sidecar及其相应的应用被认为是
追踪链的一个节点。如果使用的追踪系统支持Span类型，那么理想的追踪链如下：

```
-> [[SERVER (inbound sidecar) -> App -> CLIENT (outbound sidecar)]] -> ...
                                 App
```

在此模式中，处理入站请求的sidecar将生成SERVER类型的Span，
处理出站请求的sidecar将生成CLIENT类型的Span。应用本身不产生Span，
而只负责转发追踪上下文。

在第二种模式下，Envoy可以作为网关使用，或者虽然作为sidecar使用，
但sidecar及其应用被看作追踪链中的独立节点。如果使用的追踪系统支持Span类型，
那么理想的追踪链如下：

```
-> [SERVER -> CLIENT] -> [SERVER -> CLIENT] -> [SERVER -> CLIENT] -> [SERVER -> CLIENT] -> ...
        Gateway           Inbound Sidecar            App             Outbound Sidecar
```

此模式下，Envoy为下游请求生成SERVER类型的Span，为上游请求生成CLIENT类型的Span。
应用也可以产生自己的Span来记录其内部工作。

要启用这种模式，需要将[spawn_upstream_span](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-tracing-spawn-upstream-span)明确设置为true。
这样做会让追踪系统为上游请求生成CLIENT类型的Span，并将Envoy作为追踪链中的
一个独立节点对待。

