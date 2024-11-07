---
title: "Envoy 介绍之(一)"
date: 2023-12-11T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍", "中文文档"]
Categories: ["envoy"]
DisableComments: false
---

本文档基于 v1.28, 原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(概述、监听和HTTP部分)
<!--more-->

# 1. 什么是 Envoy

Envoy 是一个 L7 代理和通信总线，专为大型现代面向服务的架构而设计。 该项目的诞生源于以下理念：

> 网络应该对应用程序透明。当网络和应用程序出现问题时 应该很容易确定问题的根源。

在实践中，实现上述目标是非常困难的。Envoy 试图通过提供以下高级功能达成上述目标：

**进程外架构：** Envoy 是一个独立的进程，旨在与每个应用程序服务一起运行。
所有 Envoy 形成一个透明的通信网格，其中每个应用程序都向 localhost 发送和接收消息，
并且不感知网络拓扑。与传统的服务间通信库方法相比，进程外架构具有两个显著优势：

* Envoy 可以与任何应用程序语言一起使用。单个 Envoy 部署可以在 Java、C++、Go、
PHP、Python 等之间形成网格。在面向服务的架构中, 经常会使用多种应用程序框架和语言,
这正变得越来越普遍。Envoy 透明地弥合了这之间的差距。

* 任何使用过大型面向服务架构的人都知道，部署库升级可能是非常痛苦的。
Envoy 可以在整个基础架构上, 快速透明地部署和升级。

**L3/L4 过滤器架构：** 从本质上讲，Envoy 是一个 L3/L4 网络代理。envoy 设计了
可插拔的 filter 链机制, 这允许我们编写过滤器, 并把它插入到 main server 中, 
来执行不同的 TCP/UDP 代理任务, envoy 本身已经提供了一些过滤器来支持各种任务，
例如 TCP proxy, UDP proxy, HTTP proxy, TLS client certificate authentication, 
Redis, MongoDB, Postgres 等。

**HTTP L7 过滤器架构：** HTTP 是现代应用程序架构中如此重要的组成部分，
以至于 Envoy 支持 一个额外的 HTTP L7 过滤器层。HTTP 过滤器可以插入到 
HTTP 连接管理子系统中，这些过滤器执行不同任务的, 例如缓冲、限流、路由/转发、
嗅探 Amazon 的 DynamoDB 等。

**一流的 HTTP/2 支持：** 在 HTTP 模式下运行时，Envoy 支持 HTTP/1.1 和 HTTP/2。
Envoy 可以作为双向透明的 HTTP/1.1 到 HTTP/2 代理运行。这意味着可以桥接任何组合
的 HTTP/1.1 和 HTTP/2 客户端和目标服务器。推荐在envoy上的服务间配置使用 HTTP/2 
来创建持久连接网格，这样请求和响应可以在此网格上进行多路复用。

**HTTP/3 支持（目前处于 alpha 版）：** 截至 1.19.0，Envoy 现在支持 HTTP/3 上游和下游，
并可以在任何方向翻译 HTTP/1.1、HTTP/2 和 HTTP/3 之间的组合。

**HTTP L7 路由：** 在 HTTP 模式下运行时，Envoy 支持一个 routing 子系统，
该子系统可以根据path, authority, content type, 运行时值等对请求进行路由和重定向。
该功能在将 Envoy 用作前端/边缘代理时最有用，但在构建服务网格时也能使用。

**gRPC 支持：** gRPC 是来自 Google 的 RPC 框架，使用 HTTP/2 或更高版本作为底层多路传输。
Envoy 支持使用 gRPC 请求和响应所需的所有 HTTP/2 功能。这两个系统非常互补。

**服务发现和动态配置：** Envoy 可以选择使用一组分层的动态配置 API 进行集中管理。 
这些层为 Envoy 提供了有关以下内容的动态更新： 后端集群中的主机， 后端集群本身、
HTTP 路由、侦听套接字和加密材料。 为了简化部署，后端主机发现可以通过 DNS 解析
（甚至完全跳过）来完成。 其他层被静态配置文件取代。

**健康检查：** 推荐的构建 Envoy 网格方式是将服务发现视为最终一致的过程。
Envoy 包含一个健康检查子系统，可以可选地对上游服务集群执行主动健康检查。
然后，Envoy 使用服务发现和健康检查信息的并集来确定健康的负载均衡目标。
Envoy 还通过异常值检测子系统支持被动健康检查。

**高级负载均衡：** 在分布式系统中不同组件之间的 load balancing 是一个复杂的问题。
因为 Envoy 是一个独立的代理而不是一个库，它能够在一个地方实现高级负载均衡技术，
并使它们对任何应用程序都可访问。目前 Envoy 支持自动重试、熔断、通过外部速率限制服务
进行全局速率限制、request shadowing 和异常值检测。计划将来支持请求 request racing。

**前端边缘代理支持：** 在边缘使用相同的软件具有很大优势（可观察性、管理、相同的服务发现和负载均衡算法等）。
Envoy有一套特性集，使其非常适合作为大多数现代Web应用程序用例的前端代理。
这包括TLS终止、HTTP/1.1 HTTP/2和HTTP/3支持，以及HTTP L7路由。

**一流的可观察性：** 如上所述，Envoy的主要目标是使网络透明。然而，问题既发生在网络层面，也发生在应用层面。
Envoy为所有子系统提供了强大的统计支持。statsd（和兼容的提供者）是目前支持的统计接收器，
虽然插入不同的接收器并不困难。统计数据也可以通过管理端口查看。Envoy还支持通过第三方提供商进行分布式跟踪。

# 2. 架构概述

## 2.1 介绍

### 2.1.1 术语定义

在我们深入了解主要架构文档之前，先给出一些定义。一些定义在业界略有争议，
但这是 Envoy 在整个文档和代码库中使用它们的方式，所以这就是生活。

- 主机 (Host)：能够进行网络通信的实体（手机上的应用程序、服务器等）。
在本文档中，主机是逻辑网络应用程序。一个物理硬件可以有多个主机在上面运行，只要它们中的每一个都可以被独立寻址。

- 下游 (Downstream)：下游主机连接到 Envoy，发送请求并接收响应。

- 上游 (Upstream)：上游主机接收来自 Envoy 的连接和请求，并返回响应。

- 侦听器 (Listener)：侦听器是一个命名的网络位置（例如，端口、UNIX 域套接字等），
  下游客户端可以连接到它。Envoy 暴露了一个或多个侦听器，下游主机连接到它。

- 集群 (Cluster)：集群是一组逻辑上相似的上游主机，Envoy 连接到这些主机。Envoy 通过 服务发现 发现集群成员。
  它还可以通过 主动健康检查 选择性地确定集群成员的健康状况。Envoy 将请求路由到的集群成员由 负载均衡策略 决定。
  
- 网格 (Mesh)：协调一致的网络拓扑的一组主机。在本文档中，“Envoy 网格”是指一组 Envoy 代理，
  它们构成一个消息传递基底，用于由许多不同服务和应用程序平台组成的分布式系统。

- 运行时配置：Envoy外部部署的运行时实时配置系统。可以改变配置设置，从而在不重新启动Envoy或更改主配置的情况下影响操作。


### 2.1.2 线程模型

Envoy 使用单进程多线程架构。

一个主线程控制各种零散的协调任务，而一定数量的 worker 线程执行监听、过滤和转发。

一旦连接被侦听器接受，连接将在其生命周期内绑定到单个 worker 线程。这允许 Envoy 的大部分代码
都是单线程的（可并行执行），并有一小部分更复杂的代码处理 worker 线程之间的协调。

通常，Envoy 被编写为 100% 非阻塞。

> **提示:**
>
> 对于大多数工作负载，我们建议将 worker 线程的数量配置为等于机器上硬件线程的数量。


#### 2.1.2.1 侦听器连接平衡

默认情况下，worker 线程之间没有协调。这意味着所有 worker 线程都独立地尝试在每个侦听器上接受连接，并依赖内核在线程之间进行适当的平衡。

对于大多数工作负载，内核在平衡传入连接方面做得非常出色。然而，对于一些工作负载，
特别是那些具有少量非常长连接的工作负载（例如，服务网格 HTTP2/gRPC 出口），
可能需要 Envoy 强制在 worker 线程之间平衡连接。
为了支持此行为，Envoy 允许在每个侦听器上配置不同类型的连接平衡。

> **注意:**
>
> 在 Windows 上，内核无法使用 Envoy 使用的异步 I/O 模型正确地平衡连接。

在平台修复此问题之前，Envoy 将强制执行 Windows 上的侦听器连接平衡。这允许我们在不同的 worker 线程之间平衡连接。这种行为会带来性能损失。

## 2.2 监听

### 2.2.1 监听器

Envoy 配置支持单个进程中的任意数量的侦听器。通常，我们建议无论配置了多少侦听器，都只在一个机器上运行一个 Envoy。这使得操作更简单，并提供了一个统计信息的来源。

Envoy 支持 TCP 和 UDP 侦听器。

#### 2.2.1.1 TCP

每个监听器独立配置了filter_chains，根据filter_chain_match标准选择单个filter_chain。

单个filter_chain由一个或多个网络级别（L3/L4）过滤器组成。

当在监听器上接收到新连接时，选择适当的filter_chain，并实例化配置的连接本地过滤器堆栈并开始处理后续事件。

通用侦听器架构用于执行 Envoy 负责的大多数不同的代理任务（例如，限流、TLS 客户端认证、HTTP 连接管理、MongoDB 嗅探、原始 TCP 代理等）。

侦听器还可以选择性地配置一些侦听器过滤器。这些过滤器在网络级过滤器之前进行处理，并有机会操纵连接元数据，通常是为了影响连接如何被后面的过滤器或集群处理。

侦听器也可以通过侦听器发现服务 (LDS) 动态获取。

> **提示**
>
> 有关参考文档，请参见侦听器配置、protobuf 和组件部分。

#### 2.2.1.2 UDP

Envoy 除了支持 TCP 侦听器外，还支持 UDP 侦听器，并特别支持 UDP 侦听器过滤器。

UDP 侦听器过滤器在每个 worker 上实例化一次，并对该 worker 全局有效。

每个侦听器过滤器处理worker 在端口上监听时收到的每个 UDP 数据报。

实际上，UDP 侦听器使用 SO_REUSEPORT 内核选项进行配置，该选项将导致内核将每个 UDP 4 元组一致地哈希到同一个 worker 上。这允许 UDP 侦听器过滤器如果需要的话可以是“会话”导向的。此功能的一个内置示例是 UDP 代理侦听器过滤器。


### 2.2.2 监听过滤器

Envoy 的监听器过滤器可以用于操纵连接元数据。

监听器过滤器的主要目的就是, 在不更改 Envoy 核心功能的前提下, 来轻松地添加其他系统集成功能，并且还使多个这样的功能之间的交互更加明确。

监听器过滤器的 API 相对简单，因为这些过滤器最终在新的已接受套接字上运行。

过滤器链中的过滤器可以停止并随后继续给其他过滤器处理。这允许更复杂的场景，例如调用速率限制服务等。

Envoy 本身包括几个在本文档, 和配置参考文档中记录的监听过滤器。

#### 2.2.2.1 过滤器链

##### 过滤器链匹配

网络过滤器按顺序链接在 FilterChain 中。

每个监听器可以有多个 FilterChain 和一个可选的 default_filter_chain。

在接收到请求时，使用具有最特定匹配条件的 FilterChain。

如果没有找到匹配的 FilterChain，将使用配置中的 default_filter_chain 来为请求提供服务，否则将关闭连接。

##### 过滤器链单独更新

过滤器链可以独立更新。

在监听器配置更新时，如果监听器管理器确定监听器更新只是过滤器链的更新，则监听器更新将通过添加、更新和删除过滤器链来执行。

这些被销毁的过滤器链所拥有的连接将被清空，如这里所述。

如果新的过滤器链和旧的过滤器链在protobuf消息中是等效的，则相应的过滤器链运行时信息将保留。所保留的过滤器链拥有的连接将保持打开状态。

并非所有的监听器配置更新都可以通过过滤器链更新来执行。例如，如果新的监听器配置中更新了监听器元数据，新的过滤器链必须接收新的元数据。在这种情况下，整个监听器将被清空并更新。

#### 2.2.2.2 网络（L3/L4）过滤器：

网络级别（L3/L4）过滤器构成了Envoy连接处理的核心。过滤器API允许将不同组合的过滤器混合、匹配并附加到给定的侦听器上。有三种不同类型的网络过滤器：

- 读取：

  读取过滤器在Envoy从下游连接接收到数据时被调用。

- 写入：

  写入过滤器在Envoy即将向下游连接发送数据时被调用。

- 读取/写入：

  读取/写入过滤器在Envoy从下游连接接收到数据以及即将向下游连接发送数据时都被调用。

网络级别过滤器的API相对简单，因为最终这些过滤器在原始字节和一些连接事件（例如，TLS握手完成，本地或远程连接断开等）上运行。

过滤器链中的过滤器可以停止并随后继续给其他过滤器处理。这允许更复杂的场景，例如调用速率限制服务等。

网络级别过滤器还可以在单个下游连接的上下文中共享状态（静态和动态）。更多详细信息请参阅**过滤期间数据共享**部分。

> **提示**
> See the listener configuration and protobuf sections for reference documentation.
>
> See here for included filters.

##### UDP代理过滤器：

Envoy通过UDP代理监听器过滤器支持UDP代理。

##### DNS过滤器：

通过配置UDP侦听器DNS过滤器，Envoy支持响应DNS请求。DNS过滤器支持响应A和AAAA记录的前向查询。答案可以从静态配置的资源、集群或外部DNS服务器中发现。过滤器将返回大小限制为512字节的DNS响应。如果域配置了多个地址或集群具有多个端点，Envoy将返回每个发现地址，直至上述大小限制。

##### 连接限制过滤器：

Envoy通过连接限制过滤器和运行时连接限制支持本地（非分布式）L4连接限制。


## 2.3 HTTP

### 2.3.1 HTTP连接管理：

HTTP是现代面向服务架构的关键组成部分，Envoy实现了大量的HTTP特定功能。Envoy有一个内置的网络级别过滤器，称为HTTP连接管理器。

此过滤器将原始字节翻译成HTTP级别消息和事件（例如，收到标头，收到正文数据，收到 Trailer 数据等）。

它还处理所有HTTP连接和请求的常见功能，例如访问日志记录，请求ID生成和跟踪，请求/响应标头操作，路由表管理以及统计信息。

> **提示：**
>
> 请参阅HTTP连接管理器配置和protobuf部分以获取参考文档。

#### 2.3.1.1 HTTP协议：

Envoy的HTTP连接管理器对HTTP/1.1、HTTP/2和HTTP/3（包括WebSockets）具有原生支持。

Envoy的HTTP支持旨在首先成为一个HTTP/2多路复用代理。在内部，使用HTTP/2术语来描述系统组件。例如，HTTP请求和响应在“流”上进行。

使用编解码器API将不同协议转换为协议无关的形式，用于流、请求、响应等。

在HTTP/1.1的情况下，编解码器将该协议的串行/管道化功能转换为对更高级别看起来像HTTP/2的东西。这意味着大部分代码不需要了解流是在HTTP/1.1、HTTP/2还是HTTP/3连接上产生的。

#### 2.3.1.2 HTTP标头清理：

HTTP连接管理器出于安全原因执行各种标头清理操作。

#### 2.3.1.3 路由表配置：

每个HTTP连接管理器过滤器都有一个关联的路由表，可以通过以下两种方式之一指定：

- 静态方式。

- 通过 RDS API 的动态方式。

#### 2.3.1.4 重试插件配置：

在重试期间，主机选择通常遵循与原始请求相同的过程。重试插件可用于修改此行为，它们分为两类：

- 主机谓词

  这些谓词可用于“拒绝”主机，从而导致主机选择被重新尝试。可以指定任意数量的这些谓词，如果任何一个谓词拒绝主机，则该主机将被拒绝。

  Envoy支持以下内置主机谓词：

  - PreviousHostsPredicate
  
    这将跟踪以前尝试过的主机，并拒绝已经尝试过的主机。

  - OmitCanaryHostsPredicate

    这将拒绝任何标记为金丝雀主机的主机。主机通过在端点的过滤器元数据中设置envoy.lb过滤器的canary: true来标记。请参阅LbEndpoint以了解更多详细信息。

  - OmitHostMetadataConfig

    这将根据预定义的元数据匹配标准拒绝任何主机。有关详细信息，请参阅下面的配置示例。

- 优先级谓词

  这些谓词可用于调整选择优先级的优先级负载。只能指定一个优先级谓词。

  Envoy支持以下内置优先级谓词：

  - PreviousPrioritiesConfig

    这将跟踪以前尝试过的优先级，并调整优先级负载，以便在后续重试尝试中针对其他优先级进行目标选择。

主机选择将继续，直到配置的谓词接受主机或达到可配置的最大尝试次数。

这些插件可以组合起来影响主机选择和优先级负载。与添加自定义过滤器的方式类似，Envoy也可以通过添加自定义重试插件来扩展。

##### 重试配置示例：

- PreviousHostsPredicate

  例如，要配置重试以优先选择尚未尝试的主机，可以使用内置的PreviousHostsPredicate：

[retry-previous-hosts.yaml](https://www.envoyproxy.io/docs/envoy/v1.28.0/_downloads/26839fd8ace62acc309681d35dab6fdd/retry-previous-hosts.yaml)
```yaml
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: cluster_0
                  retry_policy:
                    retry_host_predicate:
                    - name: envoy.retry_host_predicates.previous_hosts
                      typed_config:
                        "@type": type.googleapis.com/envoy.extensions.retry.host.previous_hosts.v3.PreviousHostsPredicate
                    host_selection_retry_max_attempts: 3

  clusters:
```

这将拒绝先前尝试过的主机，最多重试3次主机选择。为了处理无法找到可接受主机的情况（没有主机满足谓词）或非常不可能的情况（唯一合适的主机的相对权重非常低），对尝试次数进行限制是必要的。

- OmitHostMetadataConfig

  要根据主机的元数据拒绝主机，可以使用OmitHostMetadataConfig：

[retry-omit-host-metadata.yaml](https://www.envoyproxy.io/docs/envoy/v1.28.0/_downloads/7068c28039f68b3d0e83199a72d7c98b/retry-omit-host-metadata.yaml)
```yaml
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: cluster_0
                  retry_policy:
                    retry_host_predicate:
                    - name: envoy.retry_host_predicates.omit_host_metadata
                      typed_config:
                        "@type": type.googleapis.com/envoy.extensions.retry.host.omit_host_metadata.v3.OmitHostMetadataConfig
                        metadata_match:
                          filter_metadata:
                            envoy.lb:
                              key: value

  clusters:
```

  这将拒绝其元数据中具有匹配（键，值）的任何主机。

- PreviousPrioritiesConfig

  要配置重试以在重试期间尝试其他优先级，可以使用内置的PreviousPrioritiesConfig。

[retry-previous-priorities.yaml](https://www.envoyproxy.io/docs/envoy/v1.28.0/_downloads/b3ac02aa3ae16683880f1e6bb48df66d/retry-previous-priorities.yaml)
```yaml
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: cluster_0
                  retry_policy:
                    retry_priority:
                      name: envoy.retry_priorities.previous_priorities
                      typed_config:
                        "@type": type.googleapis.com/envoy.extensions.retry.priority.previous_priorities.v3.PreviousPrioritiesConfig
                        update_frequency: 2

  clusters:
```

  这将针对后续重试尝试中尚未使用的优先级。update_frequency参数决定应重新计算优先级负载的频率。

##### 组合重试策略：

这些插件可以组合使用，这将同时排除之前尝试过的主机和之前尝试过的优先级。

[retry-combined-hosts-priorities.yaml](https://www.envoyproxy.io/docs/envoy/v1.28.0/_downloads/118b46ea90fdbc43ca1844285666eb57/retry-combined-hosts-priorities.yaml)
```yaml
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: cluster_0
                  retry_policy:
                    retry_host_predicate:
                    - name: envoy.retry_host_predicates.previous_hosts
                      typed_config:
                        "@type": type.googleapis.com/envoy.extensions.retry.host.previous_hosts.v3.PreviousHostsPredicate
                    host_selection_retry_max_attempts: 3
                    retry_priority:
                      name: envoy.retry_priorities.previous_priorities
                      typed_config:
                        "@type": type.googleapis.com/envoy.extensions.retry.priority.previous_priorities.v3.PreviousPrioritiesConfig
                        update_frequency: 2

  clusters:
```

#### 2.3.1.5 内部重定向

Envoy支持在内部处理3xx重定向，即捕获可配置的3xx重定向响应，合成新的请求，将其发送到由新路由匹配指定的上游，并将重定向的响应作为对原始请求的响应返回。原始请求的头和正文将被发送到重定向的新位置。目前还不支持 Trailer。

内部重定向通过在路由配置中的 `internal_redirect_policy` 字段进行配置。当重定向处理处于打开状态时，来自上游的任何3xx响应（与`redirect_response_codes` 字段值匹配）都受Envoy处理的重定向支配。

如果Envoy被配置为内部重定向HTTP 303并且接收HTTP 303响应，它将分发一个无正文的HTTP GET，如果原始请求不是GET或HEAD请求。否则，Envoy将保留原始的HTTP方法。有关更多信息，请参阅 [RFC 7231 第 6.4.4 节](https://tools.ietf.org/html/rfc7231#section-6.4.4)。

为了成功处理重定向，必须通过以下检查：

1. 响应代码与redirect_response_codes匹配，默认为302，或一组3xx代码（301、302、303、307、308）。

2. 具有有效、完全限定的URL的 `Location` 头。

3. 请求必须完全由Envoy处理。

4. 请求必须小于per_request_buffer_limit_bytes限制。

5. allow_cross_scheme_redirect为true（默认为false），或者下游请求的 Scheme 和 `Location` 头的 Scheme 相同。

6. 给定下游请求中已处理的内部重定向数量不超过该请求或重定向请求所触及的路由的max_internal_redirects。

7. 所有谓词都接受目标路由。

任何故障都将导致重定向传递给下游。

由于重定向请求可能会在多个路由之间反弹，因此重定向链中的任何路由，如果

- 未启用内部重定向，
- 或者当重定向链达到它时，其max_internal_redirects小于或等于重定向链的长度，
- 或者被任何谓词禁止，

则将导致重定向传递给下游。

可以使用 `previous_routes` 谓词和 `allow_listed_routes` 这两个谓词来创建一个DAG，该DAG定义了重定向链。具体而言，allow_listed_routes谓词定义了DAG中各个节点的边缘，而previous_routes谓词定义了边缘的“已访问”状态，这样如果需要，可以避免循环。

可以使用第三个谓词safe_cross_scheme来防止HTTP -> HTTPS重定向。

一旦重定向通过这些检查，将被发送到原始上游的请求标头将被修改：

- 将原始请求的完全限定URL放入x-envoy-original-url标头中。

- 用Location标头中的值替换 `Authority`/`Host`, `Scheme`, 和 `Path` 标头。

然后，将修改后的请求标头选择新的路由、通过新的过滤器链、然后与所有正常的Envoy请求一样进行过滤和发送到上游。

> **警告**
>
> 请注意，HTTP 连接管理器会进行一次性的清理，例如删除不信任的头部信息。特定路由的头部修改将会同时应用于初始和后续的路由，即便它们相同。因此，在设置头部修改规则时要特别小心，避免头部信息被重复添加。

重定向流程示例：

1. 客户端向 http://foo.com/bar 发起 GET 请求。

2. 上游服务器 1 携带 `Location: http://baz.com/eep` 头回复 302。

3. Envoy 配置为允许原路由重定向，向上游服务器 2 发起新的 GET 请求，带着额外请求头 `x-envoy-original-url: http://foo.com/bar` 去获取 http://baz.com/eep。

4. Envoy 将 http://baz.com/eep 的响应数据作为对初始请求的响应，传递给客户端。

#### 2.3.1.6 超时

HTTP 连接和流存在多种可设置的超时限制。有关重要超时设置的概览，请参考[问题条目](https://www.envoyproxy.io/docs/envoy/v1.28.0/faq/configuration/timeouts#faq-configuration-timeouts)。

#### 2.3.1.7 HTTP 头部顺序设置

Envoy 通过链表保持头部（包括伪头部，即以冒号开头的头部）的插入顺序，当头部数量较少时，这种处理方法速度非常快。

### 2.3.2 HTTP 过滤器

Envoy 在连接管理器中支持 HTTP 级别的过滤器堆栈。

可以在不了解底层物理协议（HTTP/1.1、HTTP/2 等）或复用功能的情况下，编写在 HTTP 级别消息上操作的过滤器。

HTTP 过滤器可以是下游过滤器，与给定侦听器相关联，在路由之前对每个下游请求进行流处理，也可以是上游过滤器，与给定集群相关联，在路由器过滤器之后对每个上游请求进行流处理。

有三种类型的 HTTP 级别过滤器：

- 解码器

  解码器过滤器在连接管理器解码请求流（标头、正文和尾部）的部分时被调用。

- 编码器

  编码器过滤器在连接管理器即将编码响应流（标头、正文和尾部）的部分时被调用。

- 解码器/编码器

  解码器/编码器过滤器在连接管理器解码请求流的部件时以及连接管理器即将编码响应流的部件时都被调用。

HTTP 级别过滤器的 API 允许过滤器在不了解底层协议的情况下操作。

与网络级别过滤器一样，HTTP 过滤器可以停止并继续迭代到后续过滤器。这允许更复杂的场景，如健康检查处理、调用速率限制服务、缓冲、路由、为应用程序流量生成统计信息（如 DynamoDB）等。

HTTP 级别过滤器还可以在单个请求流的上下文中共享状态（静态和动态）。有关更多详细信息，请参阅[过滤器之间的数据共享](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/advanced/data_sharing_between_filters#arch-overview-data-sharing-between-filters)。

> **提示：**
>
> 请参阅 HTTP 过滤器的配置和 protobuf 部分以获取参考文档。
> 
> 请参见[此处](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#extension-category-envoy-filters-http)以获取包含的过滤器。

#### 2.3.2.1 过滤器排序

过滤器在http_filters字段中的顺序很重要。如果过滤器按以下顺序配置（并假设所有三个过滤器都是解码器/编码器过滤器）：

```yaml
http_filters:
  - A
  - B
  # The last configured filter has to be a terminal filter, as determined by the
  # NamedHttpFilterConfigFactory::isTerminalFilterByProto(config, context) function. This is most likely the router
  # filter.
  - C
```

连接管理器将按以下顺序调用解码器过滤器：A、B、C。另一方面，连接管理器将以相反的顺序调用编码器过滤器：C、B、A。

#### 2.3.2.2 条件过滤器配置

根据传入的请求改变过滤器配置有一些支持。请参阅[复合过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/composite_filter#config-http-filters-composite)的详细信息，了解如何配置可用于给定请求的匹配树，以解析要使用的过滤器配置。

#### 2.3.2.3 过滤器路由突变

在下游HTTP过滤器链处理过程中，当 `decodeHeaders()` 被一个过滤器调用时，连接管理器执行路由解析并设置一个指向上游集群的缓存路由。

下游过滤器具有在路由解析后直接变异此缓存路由的能力，通过`setRoute`回调和`DelegatingRoute`机制。

过滤器可以创建一个`DelegatingRoute`的派生/子类来覆盖特定方法（例如，路由的超时值或路由条目的集群名称），同时保留基础路由`DelegatingRoute`的其余属性/行为。然后，可以使用`setRoute`手动将缓存路由设置为这个`DelegatingRoute`实例。此类派生类的示例可以在[ExampleDerivedDelegatingRoute](https://github.com/envoyproxy/envoy/blob/v1.28.0/test/test_common/delegating_route_utility.h)中找到。

如果没有其他过滤器在链中修改缓存路由选择（例如，过滤器通常执行的操作是清除路由缓存，setRoute将不会持久存在），此路由选择将进入路由器过滤器，该过滤器将最终确定请求将被转发到的上游集群。

#### 2.3.2.4 路由特定配置

每个过滤器配置映射可用于提供HTTP过滤器的路由或虚拟主机或路由配置特定的配置。

每个过滤器配置映射的键应与过滤器配置名称匹配。

例如，给定以下HTTP过滤器配置：

```yaml
http_filters:
- name: custom-filter-name-for-lua # Custom name be used as filter config name
  typed_config: { ... }
- name: envoy.filters.http.buffer # Canonical name be used as filter config name
  typed_config: { ... }
```

`custom-filter-name-for-lua`和`envoy.filters.http.buffer`将用作查找相关每个过滤器配置映射的键。

对于第一个`custom-filter-name-for-lua`过滤器，如果没有通过`custom-filter-name-for-lua`找到相关条目，我们将降级尝试使用规范过滤器名称`envoy.filters.http.lua`。这种降级是为了向后兼容，可以通过显式设置运行时标志`envoy.reloadable_features.no_downgrade_to_canonical_name`为`true`来禁用。

对于第二个`envoy.filters.http.buffer`过滤器，如果没有通过`envoy.filters.http.buffer`找到相关条目，我们将不会尝试降级，因为规范过滤器名称与过滤器配置名称相同。

> **警告：**
>
> 降级到规范过滤器名称已被弃用，并将很快被删除。请确保每个过滤器配置映射的键与过滤器配置名称完全匹配，不要依赖降级行为。

每个过滤器的使用每个过滤器配置映射是特定的。请参阅[HTTP过滤器文档](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/http_filters#config-http-filters)以了解如何以及在每种过滤器中使用它。

### 2.3.3 HTTP 路由

Envoy包括一个HTTP路由器过滤器，可以安装它以执行高级路由任务。

这不仅对于处理边缘流量（传统的反向代理请求处理）有用，而且对于构建Envoy网格中的服务到服务通信（通常通过在 `host`/`authority` HTTP头中进行路由以到达特定的上游服务集群）也很有用。

Envoy还具有被配置为前向代理的能力。在前向代理配置中，网格客户端可以通过适当地配置其HTTP代理来参与Envoy。

在高层面上，路由器接收传入的HTTP请求，将其与上游集群匹配，获取到上游集群中主机的连接池，并转发请求。

路由器过滤器支持许多功能，包括：

- 虚拟主机和集群

  将域/权威映射到一组路由规则。

  虚拟集群在虚拟主机级别指定，用于由Envoy生成在标准集群级别之上的额外统计信息。虚拟集群可以使用正则表达式匹配。

- 路径、前缀和头部匹配

  根据大小写敏感和前缀以及精确的请求路径进行路由，或者使用正则表达式路径匹配以及更复杂匹配规则。

  根据任意头部匹配路由。

- Path, prefix and host 重写

  使用正则表达式和捕获组重写前缀或路径。

  显式 host 重写，以及基于选定上游主机的DNS名称自动主机重写。

- 请求重定向

  在路由级别进行路径/主机重定向。

  在虚拟主机级别进行TLS重定向。

- 请求超时、重试和套利

  可以通过HTTP头或路由配置指定请求重试。

  可以通过HTTP头或路由配置指定超时。

  Envoy还为响应请求（逐个尝试）超时的重试提供了请求套利。

- 流量转移和拆分

  通过运行时代码将流量从一个上游集群转移到另一个上游集群，或者根据基于权重/百分比的路由拆分流量（参见流量转移/拆分）。

- 基于策略的路由

  基于优先级或哈希策略的路由。

- 直接响应

  在路由级别进行非代理HTTP响应。

- 绝对URLs

  支持非TLS前向代理的绝对URLs。

#### 2.3.3.1 路由范围

范围路由允许Envoy对域和路由规则的搜索空间施加约束。

`Route Scope` 将一个键与路由表相关联。

对于每个请求，通过HTTP连接管理器动态计算范围键以选择路由表。

与范围关联的 `RouteConfiguration` 可以使用 OnDemand 过滤器进行配置。

Scoped RDS（SRDS）API包含一组Scope资源，每个Scope定义独立的路由配置，以及一个 ScopeKeyBuilder 定义的键构建算法, Envoy 使用该构建算法为每个请求查找相应范围的键。

在以下静态配置的范围内，Envoy将根据分号分割Addr头值，通过等号分割它们来获取键值对，并使用找到的第一个键值对x-foo-key的值作为范围键。

具体而言，如果Addr头值为foo=1;x-foo-key=bar;x-bar-key=something-else，则计算bar作为查找相应路由配置的范围键。

[route-scope.yaml](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/http/http_routing#id1)
```yaml
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          scoped_routes:
            name: scope_by_addr
            scope_key_builder:
              fragments:
              - header_value_extractor:
                  name: Addr
                  element_separator: ";"
                  element:
                    key: x-foo-key
                    separator: "="
            scoped_route_configurations_list:
              scoped_route_configurations:
              - on_demand: true
                name: scoped_route_0
                key:
                  fragments:
                  - string_key: bar
                route_configuration:
                  name: local_route
                  virtual_hosts:
                  - name: local_service
                    domains: ["*"]
                    routes:
                    - match:
                        prefix: "/"
                      route:
                        cluster: cluster_0
```

为了使键与作用域路由配置匹配，计算键中的片段数量必须与作用域路由配置中的片段数量相匹配。然后，片段按顺序匹配。

> **注意：**
>
> 构建的键中缺少片段（视为NULL）将使请求无法匹配任何范围，即无法为请求找到任何路由条目。


#### 2.3.3.2 路由表

HTTP连接管理器配置拥有由所有配置的HTTP过滤器使用的路由表。

尽管路由器过滤器是路由表的主要使用者，但其他过滤器也可以访问它，以防它们想根据请求的最终目的地做出决策。例如，内置的速率限制过滤器会参考路由表以确定是否应该基于路由调用全局速率限制服务。

连接管理器确保对于特定请求，获取路由的所有调用都是稳定的，即使决策涉及随机性（例如在运行时配置路由规则的情况下）。

#### 2.3.3.3 重试语义

Envoy允许在路由配置以及特定请求的请求头中进行重试配置。

以下是可能的配置：

- 最大重试次数

  Envoy可以继续重试任意次数。

  重试之间的间隔由指数退避算法（默认）决定，或者基于上游服务器通过头部的反馈（如果存在）。

  > **注意**
  > 所有重试都在整个请求超时内进行。
  > 这避免了由于大量重试而导致较长的请求时间。

- 重试条件

  Envoy可以根据应用程序需求在不同的条件下进行重试。例如，网络故障，所有5xx响应代码，幂等4xx响应代码等。

- 重试预算

  Envoy可以通过重试预算限制活动请求的比例，只有这些活动请求可以进行重试，以防止造成流量剧增。

- 主机选择重试插件

  可以为Envoy配置在选择主机进行重试时应用额外的逻辑。

  指定重试主机谓词允许在选择特定主机时重新尝试主机选择，而重试优先级可以配置为调整用于重试的优先级的优先级负载。

> **注意**
> 
> Envoy在存在x-envoy-overloaded时重试请求。建议配置重试预算（首选）或将最大活动重试电路断路器设置为适当值以避免重试风暴。

#### 2.3.3.4 请求对冲

Envoy 支持请求对冲，可以通过指定对冲策略进行启用。

这意味着Envoy会并发多个上游请求，并将第一个具有可接受头的响应返回给下游。

重试策略用于确定应该返回响应还是应该等待更多响应。

目前，对冲只能在响应请求超时时执行。这意味着会发出重试请求而不会取消初始超时的请求，并等待延迟响应。根据重试策略的第一个“良好”响应将返回给下游。

这种实现确保不会对相同的上游请求进行两次重试，如果请求超时然后导致5xx响应，则可能会创建两个可重试事件，这可能会发生。

#### 2.3.3.5 优先级路由

Envoy 支持在路由级别进行优先级路由。

目前的优先级实现为每个优先级级别使用不同的连接池和电路断路设置，这意味着即使对于HTTP/2请求，也会对上游主机使用两个物理连接。

目前支持的优先级是`default`和`high `。

#### 2.3.3.5 直接响应

Envoy 支持发送“直接”响应。这些是预先配置的HTTP响应，不需要代理到上游服务器。

在Route中指定直接响应有两种方法：

- 设置direct_response字段。这对所有HTTP响应状态都有效。

- 设置redirect字段。这仅对重定向响应状态有效，但它简化了Location头部的设置。

直接响应具有HTTP状态码和可选的主体。

Route配置可以指定响应主体内联或指定包含主体的文件路径。

如果Route配置指定了文件路径名，Envoy将在配置加载时读取文件并缓存内容。

> **注意：**
> 
> 如果指定了响应主体，则默认情况下它的尺寸限制为4KB，无论它是内联提供还是在一个文件中提供。
>
> Envoy目前将整个主体保存在内存中，因此4KB的默认值旨在防止代理的内存占用变得太大。
>
> 如果需要，可以通过设置max_direct_response_body_size_bytes字段来更改此限制。

如果为Route或封闭的Virtual Host设置了response_headers_to_add，Envoy将包括指定的头部信息在直接HTTP响应中。


#### 2.3.3.6 通过通用匹配进行路由

Envoy 支持使用通用匹配树来指定路由表。

这是一个比原始匹配引擎更富有表达力的匹配引擎，能够对任意头部进行次线性匹配（不像原始匹配引擎在某些情况下只能对`:authority`进行这种操作）。

要使用通用匹配树，请在具有`Route`或`RouteList`作为动作的虚拟主机上指定一个匹配器：

[route-scope.yaml](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/http/http_routing#id2)
```yaml
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              matcher:
                matcher_tree:
                  input:
                    name: request-headers
                    typed_config:
                      "@type": type.googleapis.com/envoy.type.matcher.v3.HttpRequestHeaderMatchInput
                      header_name: :path
                  exact_match_map:
                    map:
                      "/new_endpoint/foo":
                        action:
                          name: route_foo
                          typed_config:
                            "@type": type.googleapis.com/envoy.config.route.v3.Route
                            match:
                              prefix: /foo
                            route:
                              cluster: cluster_0
                            request_headers_to_add:
                            - header:
                                key: x-route-header
                                value: new-value
                      "/new_endpoint/bar":
                        action:
                          name: route_bar
                          typed_config:
                            "@type": type.googleapis.com/envoy.config.route.v3.Route
                            match:
                              prefix: /bar
                            route:
                              cluster: cluster_1
                            request_headers_to_add:
                            - header:
                                key: x-route-header
                                value: new-value

                      "/new_endpoint/baz":
                        action:
                          name: route_list
                          typed_config:
                            "@type": type.googleapis.com/envoy.config.route.v3.RouteList
                            routes:
                            - match:
                                prefix: /baz
                                headers:
                                - name: x-match-header
                                  string_match:
                                    exact: foo
                              route:
                                cluster: cluster_2
                            - match:
                                prefix: /baz
                                headers:
                                - name: x-match-header
                                  string_match:
                                    exact: bar
                              route:
                                cluster: cluster_3

  clusters:
```

这允许使用通用匹配框架提供的额外匹配灵活性来解决用于基于路由的路由的相同Route proto消息。

> **注意**
>
> 产生的Route还指定了匹配标准。
>
> 为了达到路由匹配，除了解析路由外，还必须满足这个条件。
>
> 当使用路径重写时，匹配的路径将仅取决于解析的Route的匹配标准。
>
> 在匹配树遍历期间进行的路径匹配不会导致路径重写。

唯一支持的输入是请求标头（通过HttpRequestHeaderMatchInput）。

> **提示**
>
> 有关API整体的更多信息，请参阅[匹配API](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/advanced/matching/matching_api#arch-overview-matching-api)的相关文档。


### 2.3.4 HTTP升级

Envoy的升级支持主要用于WebSocket和CONNECT支持，但也可以用于任意升级。

升级通过HTTP过滤器链传递HTTP头和payload。

可以在配置`upgrade_configs`时选择使用或不使用自定义过滤器链。

如果只指定了`upgrade_type`，则升级头、请求和响应正文以及HTTP数据有效负载将通过默认的HTTP过滤器链传递。

为了避免对升级有效负载使用HTTP仅过滤器，可以为给定的升级类型设置自定义过滤器，直到仅使用路由器过滤器将HTTP数据发送到上游。

> **提示**
>
> 缓冲通常与升级不兼容，因此，如果默认HTTP过滤器链中配置了缓冲过滤器，则可能需要排除缓冲过滤器以进行升级，方法是使用升级过滤器，并在该列表中不包括缓冲过滤器。

可以在每个路由上启用或禁用升级。

任何按路由启用/禁用都会自动覆盖HttpConnectionManager的配置，但自定义过滤器链只能按HttpConnectionManager进行配置。

|HCM Upgrade Enabled | Route Upgrade Enabled | Upgrade Enabled |
|--------------------|-----------------------|-----------------|
|T (Default)         |T (Default)            |T                |
|T (Default)         | F                     |F                |
|F                   |T (Default)            |T                |
|F                   |F                      |F                |

> **提示**
>
> 升级的统计数据都是捆绑在一起的，因此WebSocket和其他升级的统计数据都是通过如下统计信息来追踪的，如 `downstream_cx_upgrades_total` 和`downstream_cx_upgrades_active`。


#### 2.3.4.1 跨越 HTTP/2 或 HTTP/3 的 WebSocket

HTTP/2 和 HTTP/3 支持 WebSocket, 在 envoy 里默认是关闭的. 但是Envoy确实倾向于在整个部署中使用统一的HTTP/2+网格，通过HTTP/2和更高版本对WebSocket进行隧道传输；例如，以下形式的部署：

[`Client`] —-> `HTTP/1.1` >—- [`Front Envoy`] —-> `HTTP/2` >—- [`Sidecar Envoy` —-> `HTTP/1` >—- `App`]


在这种情况下，如果客户端使用WebSocket，我们希望WebSocket能够完整地到达上游服务器，这意味着它需要经过 `HTTP/2` 这一跳。

对于HTTP/2，这是通过[扩展的CONNECT（RFC 8441）](https://www.rfc-editor.org/rfc/rfc8441)支持实现的，通过在第二层Envoy设置allow_connect为true来启用。

对于HTTP/3，有通过alpha选项allow_extended_connect配置的并行支持，因为没有正式的RFC。

WebSocket请求将被转换为HTTP/2+ CONNECT流，带有`:protocol`头指示原始升级，经过 `HTTP/2` 这一跳，并降级回 HTTP/1 WebSocket Upgrade.。

类似 `upgrade-CONNECT-upgrade` 的转换将在任何HTTP/2+跳上执行，其中存在的缺陷是HTTP/1.1方法始终假定为GET。

允许非WebSocket升级使用任何有效的HTTP方法（例如POST），当前的升级/降级机制将在最后的`Envoy-上游`这一跳, 丢弃原始方法并将升级请求转换为GET方法。

> **注意**
>
> HTTP/2+升级路径具有非常严格的HTTP/1.1合规性，因此不会代理带有正文的WebSocket升级请求或响应。


#### 2.3.4.2 `CONNECT` 支持

Envoy的`CONNECT`支持默认是关闭的（Envoy将向`CONNECT`请求发送内部生成的403响应）。

可以通过上述升级选项启用`CONNECT`支持，将升级值设置为特殊的关键词`CONNECT`。

对于HTTP/2和更高版本，CONNECT请求可能具有路径，但一般来说，对于HTTP/1.1的CONNECT请求没有路径，并且只能通过connect_matcher进行匹配。

> **注意**
>
> 当对`CONNECT`请求进行非通配符域名匹配时，`CONNECT`目标与`Host`/`Authority`头进行匹配，而不是与目标进行匹配。你可能需要包括端口（例如`hostname:port`）才能成功匹配。

Envoy可以以两种方式处理`CONNECT`，一种是像处理任何其他请求一样代理`CONNECT`头部，并让上游终止`CONNECT`请求，另一种是终止`CONNECT`请求，并将负载作为原始TCP数据转发。

当设置了`CONNECT`升级配置时，默认行为是代理`CONNECT`请求，将其视为使用升级路径的任何其他请求。

如果希望终止，可以通过设置connect_config来实现。

如果该消息存在于`CONNEC`T请求中，路由器过滤器将删除请求头，并将HTTP负载转发到上游。从上游收到初始TCP数据后，路由器将合成200响应头，然后将TCP数据作为HTTP响应体转发。

> **警告**
> 
> 如果配置不正确，此CONNECT支持模式可能会产生重大安全漏洞，因为如果它们位于正文负载中，上游将转发未经过滤的头部。
> 
> 请谨慎使用！

> **提示**
> 
> 有关代理连接的示例，请参阅configs/proxy_connect.yaml
> 
> 有关终止连接的示例，请参阅configs/terminate_http1_connect.yaml和configs/terminate_http2_connect.yaml

> **注意**
> 
> 对于通过TLS的CONNECT，Envoy目前无法配置为在单跳中以明文形式发送CONNECT请求，并在之前未加密的负载中进行加密。
> 
> 要发送明文中的CONNECT并加密负载，必须首先通过“上游”TLS回环连接转发加密的HTTP负载，然后让TCP监听器获取加密的负载并将其发送到上游。

#### 2.3.4.3 CONNECT-UDP支持

> **注意：**
> `CONNECT-UDP`目前处于alpha状态，可能还不够稳定，不建议在生产环境中使用。

`CONNECT-UDP`（[RFC 9298](https://www.rfc-editor.org/rfc/rfc9298)）允许HTTP客户端通过HTTP代理服务器创建UDP隧道。与仅限于隧道TCP的`CONNECT`不同，`CONNECT-UDP`可用于代理基于UDP的协议，如HTTP/3。

Envoy默认禁用`CONNECT-UDP`支持。类似于`CONNECT`，可以通过在[upgrade_configs](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-upgrade-configs)中设置值为特殊关键字`CONNECT-UDP`来启用它。与`CONNECT`类似，默认情况下，`CONNECT-UDP`请求将被转发到上游。必须设置connect_config以终止请求并将有效负载作为UDP数据报转发到目标。

- 配置示例

以下示例配置使Envoy将`CONNECT-UDP`请求转发到上游。请注意，upgrade_configs被设置为`CONNECT-UDP`。

[proxy_connect_udp_http3_downstream.yaml]([https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/http/upgrades#id4](https://www.envoyproxy.io/docs/envoy/v1.28.0/_downloads/f623917f467541efc447cc5c00b0dcd7/proxy_connect_udp_http3_downstream.yaml)https://www.envoyproxy.io/docs/envoy/v1.28.0/_downloads/f623917f467541efc447cc5c00b0dcd7/proxy_connect_udp_http3_downstream.yaml)
```yaml
      filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: HTTP3
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains:
              - "*"
              routes:
              - match:
                  connect_matcher:
                    {}
                route:
                  cluster: cluster_0
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          http2_protocol_options:
            allow_connect: true
          upgrade_configs:
          - upgrade_type: CONNECT-UDP
  clusters:
  - name: cluster_0
```

以下示例配置使Envoy终止`CONNECT-UDP`请求，并将UDP有效载荷发送到目标。与本示例一样，必须设置connect_config以终止`CONNECT-UDP`请求。

[terminate_http3_connect_udp.yaml](https://www.envoyproxy.io/docs/envoy/v1.28.0/_downloads/a2eeb79fccd8c55ae6a46f1042c348dd/terminate_http3_connect_udp.yaml)

```yaml
      filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: HTTP3
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains:
              - "*"
              routes:
              - match:
                  connect_matcher:
                    {}
                route:
                  cluster: service_google
                  upgrade_configs:
                  - upgrade_type: CONNECT-UDP
                    connect_config:
                      {}
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
          http2_protocol_options:
            allow_connect: true
          upgrade_configs:
          - upgrade_type: CONNECT-UDP
  clusters:
  - name: service_google
```

#### 2.3.4.4 通过 HTTP 隧道化 UDP 

> **注意：**
> 原始 UDP 隧道化尚处于 alpha 状态，可能还不足以在生产环境中稳定使用。建议谨慎使用此功能。

除了上面一节中所述的 CONNECT-UDP 终止之外，Envoy 还支持通过 HTTP CONNECT 或 HTTP POST 请求隧道化原始 UDP，即所谓的 UDP 代理监听器过滤器。默认情况下，UDP 隧道化功能处于禁用状态，可以通过设置 tunneling_config 配置进行启用。


> **注意：**
> 目前，Envoy 只支持通过 HTTP/2 流进行 UDP 隧道化。

默认情况下，tunneling_config 将对每个 UDP 会话（由 5 元组中的数据报标识）升级连接以创建 HTTP/2 流。由于此升级协议要求封装机制来保留原始数据报的边界，因此需要应用 HTTP Capsule 会话过滤器。HTTP/2 流将在上游连接上复用。

与 TCP 隧道不同，TCP 隧道可以通过交替禁用连接套接字的读取来进行下游流控制。对于 UDP 数据报，此机制不受支持。因此，当隧道化 UDP 时，从下游接收到的新的数据报要么被流传输到上游（如果上游已准备就绪），要么在 UDP 代理等待上游就绪时停止。如果上游尚未准备好（例如，等待 HTTP 响应头），则可以丢弃数据报或缓冲直到上游就绪。在这种情况下，默认情况下，下游数据报将被丢弃，除非设置了 tunneling_config 中的 buffer_options。默认缓冲限制适度，以尽量避免大量不必要的缓冲内存，但可以根据所需的使用情况对其进行调整。当上游准备就绪时，UDP 代理将首先刷新所有先前缓冲的数据报。

> **注意：**
> 如果设置了 POST，则上游流不符合 connect-udp RFC，而将变成 POST 请求。将从 post_path 字段设置标头中使用的路径，并且标头将不包含目标主机和目标端口（如 connect-udp 协议所要求的）。应该谨慎使用此选项。

- 示例配置

以下示例配置使Envoy通过升级的CONNECT-UDP请求隧道化原始UDP数据报到上游。

[raw_udp_tunneling_http2.yaml](https://www.envoyproxy.io/docs/envoy/v1.28.0/_downloads/67a360ca4aff08fa633b6d9cdf84371c/raw_udp_tunneling_http2.yaml)

```yaml
        session_filters:
        - name: envoy.filters.udp.session.http_capsule
          typed_config:
            '@type': type.googleapis.com/envoy.extensions.filters.udp.udp_proxy.session.http_capsule.v3.FilterConfig
        tunneling_config:
          # note: proxy_host supports string substitution, for example setting "%FILTER_STATE(proxy.host.key:PLAIN)%"
          # will take the target host value from the session's filter state.
          proxy_host: proxy.host.com
          # note: target_host supports string substitution, for example setting "%FILTER_STATE(target.host.key:PLAIN)%"
          # will take the target host value from the session's filter state.
          target_host: target.host.com
          # note: The target port value can be overridden per-session by setting the required port value for
          # the filter state key ``udp.connect.target_port``.
          default_target_port: 443
          retry_options:
            max_connect_attempts: 2
          buffer_options:
            max_buffered_datagrams: 1024
            max_buffered_bytes: 16384
          headers_to_add:
          - header:
              key: original_dst_port
```

#### 2.3.4.5 通过 HTTP 隧道化 TCP

Envoy 还支持通过 HTTP CONNECT 或 HTTP POST 请求隧道化原始 TCP 数据报。以下是几种使用场景。

HTTP/2+ CONNECT 可用于通过预热的安全连接进行多路复用的 TCP 代理，并分摊 TLS 握手成本。

以下是示例 SMTP 设置：

[SMTP Upstream] —> raw SMTP >— [L2 Envoy] —> SMTP tunneled over HTTP/2 CONNECT >— [L1 Envoy] —> raw SMTP >— [Client]

HTTP/1.1 CONNECT 可用于让 TCP 客户端连接到其自己的目的地，通过一个 HTTP 代理服务器（例如：不支持 HTTP/2 的公司代理）。

[HTTP Server] —> raw HTTP >— [L2 Envoy] —> HTTP tunneled over HTTP/1.1 CONNECT >— [L1 Envoy] —> raw HTTP >— [HTTP Client]

> **注意：**
> 当使用 HTTP/1 CONNECT 时，每个 TCP 客户端连接都会在 L1 和 L2 Envoy 之间建立 TCP 连接，当有选择时，最好使用 HTTP/2 或更高版本。

HTTP POST 也可用于代理多路复用的 TCP，当中间代理不支持 CONNECT 时。

以下是示例 HTTP 设置：

[TCP Server] —> raw TCP >— [L2 Envoy] —> TCP tunneled over HTTP/2 or HTTP/1.1 POST >— [Intermediate Proxies] —> HTTP/2 or HTTP/1.1 POST >— [L1 Envoy] —> raw TCP >— [TCP Client]

> **提示：**
> 这种设置的示例可以在Envoy示例配置目录中找到。
> 
> 对于HTTP/1.1 CONNECT，请尝试以下任一命令：
>
> ```yaml
> envoy -c configs/encapsulate_in_http1_connect.yaml --base-id 1
> envoy -c configs/terminate_http1_connect.yaml --base-id 1
> ```
>
> 对于HTTP/2 CONNECT，请尝试以下任一命令：
>
> ```yaml
> envoy -c configs/encapsulate_in_http2_connect.yaml --base-id 1
> envoy -c configs/terminate_http2_connect.yaml --base-id 1
> ```
> 
> 对于HTTP/2 POST，请尝试以下任一命令：
>
> ```yaml
> envoy -c configs/encapsulate_in_http2_post.yaml --base-id 1
> envoy -c configs/terminate_http2_post.yaml --base-id 1
> ```
> 
> 在所有情况下，您将运行第一个Envoy，它将在端口10000上监听TCP流量，并将其封装在HTTP CONNECT或HTTP POST请求中；还有一个在10001端口上监听，去掉CONNECT头部（对POST请求不需要），并将原始TCP上游转发到google.com。

Envoy会在HTTP隧道建立成功（即CONNECT请求成功响应收到）后，才开始将下游TCP数据流传输到上游。

如果您想解封装CONNECT请求，并对解封装的有效负载进行HTTP处理，最简单的方法是使用内部侦听器。

### 2.3.5 HTTP 动态转发代理

通过结合 [HTTP 过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/http/http_filters/dynamic_forward_proxy_filter#config-http-filters-dynamic-forward-proxy)
和[自定义集群](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/clusters/dynamic_forward_proxy/v3/cluster.proto#envoy-v3-api-msg-extensions-clusters-dynamic-forward-proxy-v3-clusterconfig)，Envoy 支持 HTTP 动态转发代理。

这意味着 Envoy 可以在事先了解所有已配置的 DNS 地址的情况下执行 HTTP 代理的角色，同时仍保留 Envoy 的绝大多数优势，包括异步 DNS 解析。

实现方式如下：

- 如果目标 DNS 主机尚未在缓存中，则使用动态转发代理 HTTP 过滤器来暂停请求。
- Envoy 将开始异步解析 DNS 地址，在解析完成后恢复之前被暂停的请求。
- 任何未来的请求都不会被阻止，因为 DNS 地址已在缓存中。解析过程的工作方式类似于[逻辑 DNS](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/upstream/service_discovery#arch-overview-service-discovery-types-logical-dns) 服务发现类型，在任何给定时间都会记住单个目标地址。
- 所有已知主机都存储在动态转发代理集群中，以便它们可以显示在 [admin 输出](https://www.envoyproxy.io/docs/envoy/v1.28.7/operations/admin#operations-admin-interface)中。
- 特殊的负载均衡器将在转发过程中根据 HTTP `host`/`authorityHeader` 选择使用正确的主机。
- 一段时间内未使用的主机将受到生存时间策略(Time to Live, TTL) 的约束，并被清除。(注： 默认5m, 参考[extensions.common.dynamic_forward_proxy.v3.DnsCacheConfig](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/common/dynamic_forward_proxy/v3/dns_cache.proto#envoy-v3-api-msg-extensions-common-dynamic-forward-proxy-v3-dnscacheconfig))
- 当上游集群配置了 TLS 上下文时，Envoy 将自动对解析的主机名执行 SAN 验证并通过 SNI 指定主机名

上述实现细节意味着在稳定状态下，Envoy 可以转发大量 HTTP 代理流量，而所有 DNS 解析都在后台异步进行。

此外，所有其他 Envoy 过滤器和扩展都可以与动态转发代理支持结合使用，包括身份验证、RBAC、速率限制等。


> **提示：**
> 有关更多配置信息，请参阅[HTTP 过滤器配置文档](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/http/http_filters/dynamic_forward_proxy_filter#config-http-filters-dynamic-forward-proxy)。

#### 内存使用情况详细信息

Envoy 动态正向代理支持的内存使用详情如下：

- 每个解析的主机/端口对都使用固定数量的服务器全局内存，并在所有工作线程之间共享。
- 地址更改使用读/写锁内联执行，不需要为主机对象重新分配内存。
- 主机地址在没有活动连接引用它们后，会通过 TTL 机制清楚并回收使用的内存。
- 可以通过设置max_hosts字段来限制DNS缓存可以存储的主机数量。
- 可以通过集群的max_pending_requests熔断器来限制等待DNS缓存加载主机的请求数量。
- 对于上游长连接，即使连接仍处于打开状态，长连接拥有的底层逻辑主机也可能因为 TTL 机制到期而被移除。

上游请求和连接仍然受到其他集群熔断器（如[max_requests](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/cluster/v3/circuit_breaker.proto#envoy-v3-api-field-config-cluster-v3-circuitbreakers-thresholds-max-requests)）的限制。

目前假设共享连接之间的主机数据占用的内存相对较少，与连接和请求本身相比，不值得单独控制。

### 2.3.6 HTTP/3 概述


> **警告：**
> 虽然 HTTP/3下游支持已准备好用于生产，但仍在不断改进，可在 [area-quic](https://github.com/envoyproxy/envoy/labels/area%2Fquic) 标签中进行跟踪。
>
> HTTP/3上游支持对于本地控制网络来说很好，但还不适合一般互联网使用，并且缺少一些关键的延迟功能。请参阅下面的详细信息。

#### HTTP/3 下游

可以通过添加 [quic_options](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/udp_listener_config.proto#envoy-v3-api-field-config-listener-v3-udplistenerconfig-quic-options)、确保下游传输套接字是 [QuicDownstreamTransport](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/transport_sockets/quic/v3/quic_transport.proto#envoy-v3-api-msg-extensions-transport-sockets-quic-v3-quicdownstreamtransport) 以及将编解码器设置为 HTTP/3来启动下游 Envoy HTTP/3 支持 。

> **注意:**
> HTTP/3 尚未能完美处理热重启。


> **提示:**
> 有关示例配置，请参阅[下游 HTTP/3 配置](https://github.com/envoyproxy/envoy/blob/v1.28.7//configs/envoyproxy_io_proxy_http3_downstream.yaml)。
>
> 此示例配置包括 TCP 和 UDP 侦听器，并且 TCP 侦听器通过`alt-svc`标头宣告 HTTP/3 支持。
>
> 默认情况下，示例配置使用内核 UDP 支持，但**如果 Envoy 使用多个工作线程运行，则强烈建议使用BPF来提高生产性能 。**

##### HTTP/3 宣告

对于内部部署，如果已经明确配置了HTTP/3，则不需要宣告HTTP/3，但是对于面向互联网的部署，由于默认使用TCP，客户端（如Chrome）只有在明确宣告HTTP/3的情况下才会尝试使用HTTP/3。

##### BPF 使用

如果配置了多个工作线程，Envoy 将默认尝试在 Linux 上使用 BPF，但可能需要 root 权限，或者至少sudo具有权限（例如 `sudo setcap cap_bpf+ep`）。

如果配置了多个工作线程并且平台不支持 BPF，或者尝试失败，Envoy 将在启动时记录警告。

##### 下游统计数据

建议监控一些 UDP 监听器和 QUIC 连接统计数据：

- [UDP 侦听器downstream_rx_datagram_dropped](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/listeners/stats#config-listener-stats-udp)

  非零表示内核的 UDP 侦听套接字的接收缓冲区不够大。在 Linux 中，可以通过在级别 设置预绑定套接字选项，通过侦听器socket_options进行配置。SO_RCVBUFSOL_SOCKET

- [QUIC 连接错误代码和流重置错误代码](https://github.com/envoyproxy/envoy/blob/v1.28.7/config_http_conn_man_stats_per_listener_http3)

  有关每个错误代码的含义，请参阅[quic_error_codes.h](https://github.com/google/quiche/blob/main/quiche/quic/core/quic_error_codes.h) 。

#### HTTP/3 上游

实现了 HTTP/3 上游支持，既支持显式 HTTP/3（用于数据中心），也支持自动 HTTP/3（用于互联网）。

如果您处于受控环境中，UDP 不太可能被阻止，则可以在[http_protocol_options](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto#envoy-v3-api-msg-extensions-upstreams-http-v3-httpprotocoloptions)中将其配置为显式协议。

对于互联网使用，通过 [http3_protocol_options](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto#envoy-v3-api-field-extensions-upstreams-http-v3-httpprotocoloptions-explicithttpconfig-http3-protocol-options) 
配置 
[auto_config](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto#envoy-v3-api-field-extensions-upstreams-http-v3-httpprotocoloptions-auto-config) 
将导致 Envoy 尝试对那些通过 `alt-svc` 头部明确宣布支持 HTTP/3 的端点使用 HTTP/3。

当使用带有 
[http3_protocol_options](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto#envoy-v3-api-field-extensions-upstreams-http-v3-httpprotocoloptions-explicithttpconfig-http3-protocol-options) 
的 
[auto_config](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto#envoy-v3-api-field-extensions-upstreams-http-v3-httpprotocoloptions-auto-config) 
时，Envoy 将尝试创建一个 QUIC 连接，然后如果 QUIC 握手在短暂的延迟后仍未完成，将启动一个 TCP 连接，并使用最先建立的连接。

> **提示**
>
> 有关 HTTP/3 连接池的更多信息，请参见[此处](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/upstream/connection_pooling#arch-overview-http3-pooling-upstream)，包括 QUIC 将被使用的详细情况，以及当配置为可选使用 QUIC 时如何回退到 TCP。
>
> 可以在[此处](https://github.com/envoyproxy/envoy/blob/v1.28.7//configs/google_com_http3_upstream_proxy.yaml)找到上游 HTTP/3 配置文件的示例。
