---
title: "Envoy 介绍"
date: 2023-12-11T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---


Envoy 官网介绍文档的中文翻译, 原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro
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
