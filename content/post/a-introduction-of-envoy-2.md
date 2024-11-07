---
title: "Envoy 介绍之(二)"
date: 2023-12-18T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍", "中文文档"]
Categories: ["envoy"]
DisableComments: false
---


本文档基于 v1.28, 原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(上游集群部分):集群管理器、服务发现、DNS解析、健康检查、连接池、负载均衡
<!--more-->

# 上游集群

## 集群管理器

### 集群管理器

Envoy的集群管理器负责管理所有配置的上游集群。与Envoy的配置可以包含任意数量的监听器一样，配置也可以包含任意数量的独立配置的上游集群。

上游集群和主机从网络/HTTP过滤器堆栈中抽象出来，因为上游集群和主机可以用于任何数量的代理任务。
集群管理器向过滤器堆栈暴露API，允许过滤器获得到上游集群的L3/L4连接，或者到上游集群的抽象HTTP连接池
的句柄（上游主机是否支持HTTP/1.1、HTTP/2或HTTP/3被隐藏）。过滤器阶段确定它需要L3/L4连接还是新的HTTP流，
集群管理器处理所有关于哪个主机可用且健康、负载均衡、线程本地存储上游连接数据（因为大部分Envoy代码都是单线程编写的）、
上游连接类型（TCP/IP、UDS）、上游协议（如果适用的话）（HTTP/1.1、HTTP/2、HTTP/3）等的复杂性。

集群管理器所知道的集群可以通过静态方式配置，或者通过集群发现服务（CDS）API动态获取配置。
动态集群获取允许将更多配置存储在中央配置服务器中，因此需要更少的Envoy重新启动和配置分发。

集群管理器[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cluster_manager#config-cluster-manager)。

CDS[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cds#config-cluster-manager-cds)。

### 集群预热

当在服务器启动时以及通过CDS进行配置时，集群都会被“预热”。这意味着，在以下操作发生之前，集群是不可用的。

- 初始服务发现负载（例如，DNS解析、EDS更新等）。

- 如果配置了主动健康检查，则初始主动健康检查通过。Envoy将向每个发现的主机发送健康检查请求，以确定其初始健康状态。

上述项目确保在Envoy开始使用它进行流量服务之前，它对集群有一个准确的了解。

在讨论集群预热时，集群"变得可用"意味着：

- 对于新添加的集群，该集群在预热之前不会在Envoy的其他部分中显示存在。也就是说，引用该集群的HTTP路由将导致404或503（取决于配置）。

- 对于更新的集群，旧的集群将继续存在并用于服务流量。当新的集群被预热时，它将与旧的集群进行原子交换，以便不会发生任何流量中断。

## 服务发现

当在配置中定义上游集群时，Envoy需要知道如何解析集群的成员。这就是所谓的服务发现。

### 支持的服务发现类型

#### 静态

静态是服务发现的最简单类型。配置显式指定每个上游主机的解析网络名称（IP地址/端口、unix域套接字等）。

#### 严格 DNS

当使用严格DNS服务发现时，Envoy将连续且异步解析指定的DNS目标。
DNS结果中的每个返回IP地址都将被视为上游集群中的一个明确的主机。
这意味着，如果查询返回三个IP地址，Envoy将假设集群中有三个主机，并且所有这三个主机都应该进行负载均衡。
如果从结果中移除一个主机，Envoy将假设该主机不再存在，并将从任何现有的连接池中排空流量。
因此，如果成功的DNS解析返回0个主机，Envoy将假设该集群没有主机。
请注意，Envoy从未在转发路径中同步解析DNS。
以最终一致性为代价，从来不用担心长时间运行的DNS查询会阻塞。

对于同一个DNS名称解析为同一IP的单个重复IP，这些IP将被去重。

对于解析为同一IP的多个DNS名称，健康检查将不会共享。
这意味着，如果使用DNS名称解析到相同IP的主动健康检查：如果一个IP在多个DNS名称之间重复很多次，可能会导致上游主机上的不必要的负载。

这段描述是关于Envoy的DNS服务发现配置。
当`respect_dns_ttl`被启用时，DNS记录的TTL（生存时间）和`dns_refresh_rate`被用来控制DNS刷新率。
对于严格的DNS集群，如果所有记录TTL的最小值为0，那么`dns_refresh_rate`将用作集群的DNS刷新率。
如果没有指定`dns_refresh_rate`，其默认值为5000ms。
`dns_failure_refresh_rate`在故障期间控制刷新频率，如果没有配置，将使用DNS刷新率。

DNS解析会发出集群统计字段`update_attempt`、`update_success`和`update_failure`。

#### 逻辑DNS

逻辑DNS使用与严格DNS类似的异步解析机制。
然而，与严格DNS不同，逻辑DNS集群仅在需要建立新连接时使用DNS查询返回的第一个IP地址。
因此，单个逻辑连接池可能包含与各种不同上游主机的物理连接。
连接永远不会被排空，即使在成功返回0个主机的DNS解析也是如此。

这种服务发现类型适用于必须通过DNS访问的大型Web服务。
这些服务通常使用round robin DNS返回许多不同的IP地址。
对于每个查询，通常返回不同的结果。如果在此场景中使用严格DNS，
Envoy将假定在每个解析间隔期间集群的成员都在变化，从而导致连接池排空、连接循环等。
相反，使用逻辑DNS时，连接保持活动状态，直到它们被循环。
在与大型Web服务交互时，这是所有可能的世界中最好的：异步/最终一致的DNS解析、长生命周期的连接和转发路径中的零阻塞。

如果启用respect_dns_ttl，则使用DNS记录的TTL和dns_refresh_rate来控制DNS刷新率。
对于逻辑DNS集群，如果第一个记录的TTL为0，则将使用dns_refresh_rate作为集群的DNS刷新率。
如果不指定dns_refresh_rate，则默认为5000毫秒。dns_failure_refresh_rate控制故障期间的刷新频率，如果没有配置，将使用DNS刷新率。

DNS解析会发出更新尝试、更新成功和更新失败的集群统计字段。

#### 原始目标

当通过iptables REDIRECT或TPROXY目标或使用Proxy Protocol将传入连接重定向到Envoy时，可以使用原始目标集群。
在这些情况下，被重定向到原始目标集群的请求会根据重定向元数据中的地址转发给上游主机，而无需进行任何显式的host配置或上游主机发现。
对上游主机的连接池化和未使用的主机会在不再被任何连接池使用超过cleanup_interval（默认值为5000ms）时被清除。
如果原始目标地址不可用，则不会打开上游连接。Envoy还可以从HTTP头中获取原始目标。
原始目标服务发现必须与原始目标负载均衡器一起使用。
当将原始目标集群用于HTTP上游时，请将`idle_timeout`设置为5分钟，以限制上游HTTP连接的持续时间。

### Endpoint发现服务（EDS）

`Endpoint发现服务`是一个基于gRPC或REST-JSON API的服务器，由Envoy使用来获取集群成员。
在Envoy术语中，集群成员被称为“endpoint”。
对于每个集群，Envoy从发现服务中获取端点。
EDS是首选的服务发现机制，原因有以下几点：

- Envoy对每个上游主机有明确的了解（相对于通过DNS解析的负载均衡器进行路由），并且可以做出更智能的负载均衡决策。

- 在每个主机的发现API响应中携带的额外属性向Envoy提供了主机的负载均衡权重、金丝雀状态、区域等。
这些附加属性由Envoy网格在负载均衡、统计收集等方面全局使用。

Envoy项目提供了EDS和其他发现服务的参考gRPC实现，包括Java和Go的实现。

#### 自定义集群

Envoy还支持自定义集群发现机制。使用cluster_type字段在集群配置中指定自定义集群。

通常，活动健康检查与最终一致的服务发现服务数据一起使用，以进行负载平衡和路由决策。这一点将在下一节中进一步讨论。

### 最终一致的服务发现

许多现有的RPC系统将服务发现视为一个完全一致的过程。
为此，它们使用完全一致的领导者选举后端存储，如Zookeeper、etcd、Consul等。
根据我们的经验，大规模运营这些后端存储是痛苦的。

Envoy从一开始就被设计为不需要完全一致的服务发现。
相反，Envoy假设主机以最终一致的方式进出网格。

我们推荐的部署服务到服务Envoy网格配置的方法是使用最终一致的服务发现以及主动健康检查
（Envoy显式地对上游集群成员进行健康检查）来确定集群健康状况。这种范式具有以下优点：

- 所有健康决策都是完全分布式的。因此，网络分区被优雅地处理（应用程序是否优雅地处理分区是另一个故事）。

- 当为上游集群配置健康检查时，Envoy使用2x2矩阵来确定是否路由到主机：

| Discovery Status | Health Check OK | Health Check Failed  |
| ---------------- | --------------- | -------------------- |
| Discovered       | Route           | Don’t Route          |
| Absent           | Route           | Don’t Route / Delete |


**主机已发现/健康检查正常**

Envoy将路由到目标主机。

**主机缺失/健康检查正常：**

Envoy将路由到目标主机。这非常重要，因为设计假设发现服务可能在任何时候失败。
即使在发现数据中缺失主机后，如果主机继续通过健康检查，Envoy仍将路由。
虽然在此情况下无法添加新主机，但现有主机将继续正常运行。
当发现服务正常操作时，数据最终将重新收敛。

**主机已发现/健康检查失败**

Envoy不会路由到目标主机。健康检查数据被认为比发现数据更准确。

**主机缺失/健康检查失败**

Envoy不会路由并删除目标主机。

这是Envoy会清除主机数据的唯一状态。


## DNS解析

许多Envoy组件都会进行DNS解析：不同的集群类型（strict dns，logical dns）；动态转发代理系统（由一个集群和一个过滤器组成）；udp dns过滤器等。Envoy使用[c-ares](https://github.com/c-ares/c-ares)作为第三方DNS解析库。在Apple操作系统上，Envoy还通过envoy.restart_features.use_apple_api_for_dns_lookups运行时特性提供了使用Apple特定API的解析。

Envoy通过扩展提供DNS解析，并包含3个内置扩展：

1. c-ares：CaresDnsResolverConfig

2. Apple（仅限iOS/macOS）：AppleDnsResolverConfig

3. getaddrinfo：GetAddrInfoDnsResolverConfig

有关内置DNS类型配置的示例，请参阅[HTTP过滤器配置文档](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/dynamic_forward_proxy_filter#config-http-filters-dynamic-forward-proxy)。

基于`c-ares`的DNS解析器在`dns.cares`统计树中发出以下统计信息：

| Name | Type | Description |
| --- | --- | --- |
| resolve_total | Count | DNS查询的总数 |
| pending_resolutions | Gauge | 待处理的DNS查询的数量 |
| not_found | Counter | 返回NXDOMAIN或NODATA响应的DNS查询数量 |
| timeout | Counter | 超时的DNS查询数量 |
| get_addr_failure | Counter | DNS查询过程中发生的常规故障数量 |

基于Apple的DNS解析器在`dns.apple`统计树中发出以下统计信息：

| Name | Type | Description |
| :---: | :---: | :---: |
| connection_failure | Counter | 连接到DNS服务器的失败尝试的数量 |
| get_addr_failure | Counter | 调用GetAddrInfo API时发生的一般故障数量 |
| network_failure | Counter | 由于网络连接问题导致的故障数量 |
| processing_failure | Counter | 处理来自DNS服务器数据时发生的故障数量 |
| socket_failure | Counter | 获取到DNS服务器套接字的文件描述符的失败尝试的数量 |
| timeout | Counter | 超时的查询的数量 |


## 健康检查

Envoy支持针对每个上游集群进行主动健康检查的配置。正如在服务发现部分所述，主动健康检查与EDS服务发现类型是相辅相成的。然而，即使在使用其他服务发现类型时，也可能需要主动健康检查。Envoy提供了三种不同类型的健康检查以及各种设置（检查间隔、在标记主机为不健康之前需要的失败次数、在标记主机为健康之前需要的成功次数等）：

* HTTP：在HTTP健康检查期间，Envoy将向上游主机发送HTTP请求。默认情况下，如果主机被认为是健康的，它期望得到200的响应。预期和可重试的响应代码是可以配置的。上游主机可以通过返回非预期或不可重试的状态码（默认情况下任何非200码）来通知下游主机不再向其转发流量。
* gRPC：在gRPC健康检查期间，Envoy将向上游主机发送gRPC请求。默认情况下，如果主机被认为是健康的，它期望得到200的响应。gRPC健康检查在此处是可配置的。
* L3/L4：在L3/L4健康检查期间，Envoy将向上游主机发送可配置的字节缓冲区。如果主机被认为是健康的，它期望字节缓冲区在响应中被回显。Envoy还支持仅L3/L4连接的健康检查。
* Redis：Envoy将发送Redis PING命令并期望PONG响应。上游Redis服务器可以用除PONG以外的任何内容进行响应，以导致立即的活动健康检查失败。可选地，Envoy可以在用户指定的键上执行EXISTS。如果键不存在，则认为通过了健康检查。这允许用户通过将指定键设置为任何值并等待流量排空来标记Redis实例进行维护。参见[redis_key](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/health_checkers/redis/v3/redis.proto#envoy-v3-api-msg-extensions-health-checkers-redis-v3-redis)。
* Thrift：Envoy将发送Thrift请求，并期望得到成功的响应。如果上游主机返回异常，健康检查将失败。详情请参见[thrift](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/health_checkers/thrift/v3/thrift.proto#envoy-v3-api-msg-extensions-health-checkers-thrift-v3-thrift)部分。

健康检查通过为集群指定的传输套接字进行。这意味着，如果一个集群使用启用了TLS的传输套接字，健康检查也将通过TLS进行。此外，可以为健康检查连接配置TLS选项，这在相应的上游使用基于ALPN的FilterChainMatch，健康检查与数据连接使用不同协议时非常有用。

### 针对每个集群成员的健康检查配置

如果已为上游集群配置了主动健康检查，可以通过设置[ClusterLoadAssignment](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/endpoint/v3/endpoint.proto#envoy-v3-api-msg-config-endpoint-v3-clusterloadassignment)中每个定义的LocalityLbEndpoints中的LbEndpoint的Endpoint中的HealthCheckConfig来为每个注册成员指定特定附加配置。

例如，要为集群成员设置备用健康检查地址和端口，可以按照以下方式配置健康检查：

```yaml
load_assignment:
  endpoints:
  - lb_endpoints:
    - endpoint:
        health_check_config:
          port_value: 8080
          address:
            socket_address:
              address: 127.0.0.1
              port_value: 80
        address:
          socket_address:
            address: localhost
            port_value: 80
```

### 健康检查事件日志

Envoy可以通过在HealthCheck配置中指定事件日志文件路径event_log_path，生成一个可选的健康检查器日志，记录弹出和添加事件。日志结构化为HealthCheckEvent消息的JSON转储。

注意：HealthCheck配置中的event_log_path已被废弃，取而代之的是使用HealthCheck event_logger 扩展。event_log_path在文件接收器扩展中用于JSON转储。

创建了一个新的事件接收器扩展目录`envoy.health_check.event_sinks`，API可以在[这里](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/health_check_event_sinks/health_check_event_sinks#api-v3-health-check-event-sinks)找到。

通过将`always_log_health_check_failures`标志设置为true，可以将Envoy配置为记录所有健康检查失败事件。

### 被动健康检查

Envoy还通过离群检测支持被动健康检查。

### 连接池交互

更多信息请参见[此处](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/connection_pooling#arch-overview-conn-pool-health-checking)。

### HTTP健康检查过滤器

当在集群之间部署Envoy网格时启用健康检查，可能会生成大量健康检查流量。Envoy包括一个HTTP健康检查过滤器，可以在配置的HTTP监听器中安装。该过滤器具有几种不同的操作模式：

- **无直通**：在此模式下，健康检查请求从未传递给本地服务。Envoy将根据服务器的当前排空状态返回200或503。

- **无直通，从上游集群健康状态计算**：在此模式下，健康检查过滤器将根据一个或多个上游集群中至少有一定比例的服务器（健康 + 降级）是否可用，返回200或503。 （如果Envoy服务器处于排空状态，则无论上游集群的健康状况如何，都将返回503。）

- **直通**：在此模式下，Envoy将每个健康检查请求传递给本地服务。服务应根据其健康状态返回200或503。

- **带缓存的直通**：在此模式下，Envoy将健康检查请求传递给本地服务，但随后在一段时间内缓存结果。随后的健康检查请求将返回在缓存时间内可用的缓存值。当缓存时间到达时，下一个健康检查请求将传递给本地服务。这是在操作大型网格时的推荐操作模式。Envoy使用持久连接进行健康检查流量，并且健康检查请求对Envoy本身的成本非常低。因此，这种操作模式提供了每个上游主机的健康状态的最终一致视图，而不会因大量健康检查请求而使本地服务不堪重负。

进一步阅读：

- 健康检查过滤器[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/health_check_filter#config-http-filters-health-check)。

- [/healthcheck/fail](https://www.envoyproxy.io/docs/envoy/v1.28.0/operations/admin#operations-admin-interface-healthcheck-fail) 管理终端节点。

- [/healthcheck/ok](https://www.envoyproxy.io/docs/envoy/v1.28.0/operations/admin#operations-admin-interface-healthcheck-ok) 管理终端节点。

### 活动健康检查快速失败

在主动健康检查与被动健康检查（异常检测）结合使用时，通常会使用较长的健康检查间隔以避免产生大量主动健康检查流量。在这种情况下，仍然希望能够通过[/healthcheck/fail](https://www.envoyproxy.io/docs/envoy/v1.28.0/operations/admin#operations-admin-interface-healthcheck-fail)管理终端节点快速排出上游主机。为了支持这一功能，路由器过滤器和HTTP主动健康检查器将对[x-envoy-immediate-health-check-fail](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/router_filter#config-http-filters-router-x-envoy-immediate-health-check-fail)头进行响应。如果此头由上游主机设置，Envoy将立即将主机标记为活动健康检查失败，并将其排除在负载均衡之外。请注意，这仅在主机的集群已配置主动健康检查时发生。健康检查过滤器将在通过/healthcheck/fail管理终端节点标记为失败时自动设置此头。

### 健康检查标识

仅仅验证上游主机对特定健康检查URL的响应并不一定意味着上游主机是有效的。例如，在云自动缩放或容器环境中使用最终一致的服务发现时，主机可能会消失，然后以相同的IP地址但是不同的主机类型重新出现。解决此问题的一种方法是每个服务类型都有一个不同的HTTP健康检查URL。该方法的缺点是，随着每个健康检查URL都是完全自定义的，整体配置变得更加复杂。

Envoy HTTP健康检查器支持[service_name_matcher](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/health_check.proto#envoy-v3-api-field-config-core-v3-healthcheck-httphealthcheck-service-name-matcher)选项。如果设置了此选项，健康检查器还会比较*x-envoy-upstream-healthchecked-cluster*响应头与*service_name_matcher*的值。如果值不匹配，则健康检查不通过。上游健康检查过滤器将在响应头中追加*x-envoy-upstream-healthchecked-cluster*。追加的值由命令行选项`--service-cluster`确定。

### 健康降级

在使用HTTP健康检查器时，上游主机可以返回`x-envoy-degraded`，以通知健康检查器该主机已降级。有关这如何影响负载均衡，请参见[这里](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/load_balancing/degraded#arch-overview-load-balancing-degraded)。



## 连接池

Envoy针对HTTP流量提供了一种抽象的连接池机制，这些机制是基于底层的线路协议（HTTP/1.1、HTTP/2、HTTP/3）实现的。
利用这些过滤器的代码无需关心底层协议是否具备多路复用的能力。在实践中，这些底层实现主要具备以下特性：

### HTTP/1.1

HTTP/1.1连接池会根据需要向上游主机申请连接，直至触达熔断限制。请求将被分配到变得可用的连接上，无论是因为某个连接已经处理完之前的请求，还是因为一个新的连接已经准备好接受首个请求。HTTP/1.1连接池不实行请求管道化，这样当上游连接中断时，只有一个下游请求需要被重置。

### HTTP/2

HTTP/2连接池能够在单一连接上多路复用多个请求，直至触及[最大并发流限制](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-quicprotocoloptions-max-concurrent-streams)和[每连接请求上限](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-max-requests-per-connection)。为了处理请求，HTTP/2连接池将建立必要数量的连接。在没有额外限制的情况下，这通常意味着只需一个连接。当收到GOAWAY帧或连接达到[每连接请求上限](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-max-requests-per-connection)时，连接池会逐渐关闭受影响的连接。一旦连接达到其[并发流的最大限制](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-quicprotocoloptions-max-concurrent-streams)，该连接将被视为忙碌状态，直到释放出流资源。只有当有待处理的请求且没有可用的连接时，系统才会建立新的连接（但不超过连接的熔断器限制）。作为反向代理的Envoy更倾向于使用HTTP/2协议，因为连接几乎不会中断。

### HTTP/3

HTTP/3连接池能够在单一连接上多路复用多个请求，直至触及[最大并发流限制](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-quicprotocoloptions-max-concurrent-streams)和[每连接请求上限](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-max-requests-per-connection)。为了处理请求，HTTP/3连接池将建立必要数量的连接。在没有额外限制的情况下，这通常意味着只需一个连接。当收到GOAWAY帧或连接达到[每连接请求上限](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-max-requests-per-connection)时，连接池会逐渐关闭受影响的连接。一旦连接达到其[并发流的最大限制](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-quicprotocoloptions-max-concurrent-streams)，该连接将被视为忙碌状态，直到释放出流资源。只有当有待处理的请求且没有可用的连接时，系统才会建立新的连接（但不超过连接的熔断器限制）。

### 自动协议选择

对于充当前向代理的Envoy，推荐使用[AutoHttpConfig](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto#envoy-v3-api-msg-extensions-upstreams-http-v3-httpprotocoloptions-autohttpconfig)进行配置，该配置通过[http_protocol_options](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto#envoy-v3-api-msg-extensions-upstreams-http-v3-httpprotocoloptions)
设定。默认它会利用TCP和ALPN选择HTTP/2和HTTP/1.1中最适合的协议。

对于集成了HTTP/3的自动HTTP配置，需要通过[alternate_protocols_cache_options](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/upstreams/http/v3/http_protocol_options.proto#envoy-v3-api-field-extensions-upstreams-http-v3-httpprotocoloptions-autohttpconfig-alternate-protocols-cache-options)设定备用协议缓
存。HTTP/3连接将仅尝试那些宣传支持HTTP/3的服务器，这种支持可能是通过[HTTP备用服务](https://tools.ietf.org/html/rfc7838)、[HTTPS
DNS资源记录](https://datatracker.ietf.org/doc/html/draft-ietf-dnsop-svcb-https-04)，或将来手动配置的“QUIC提示”来表明的。如果没有这样的宣传，则会转而使用HTTP/2或HTTP/1。

在尝试使用HTTP/3时，Envoy会优先尝试建立QUIC连接。如果300毫秒内QUIC连接未建立，Envoy将同时
也会尝试建立TCP连接。无论哪种握手成功，都会被用来初始化流。但若TCP和QUIC连接都成功建立，
系统将最终优先选择QUIC。

由于HTTP/3运行于QUIC（基于UDP）之上，而非TCP（HTTP/1和HTTP/2采用的协议），网络设备屏蔽
UDP流量的情况相对常见，这也可能导致HTTP/3被封锁。因此，上游HTTP/3的连接尝试可能会遭遇网络
阻断，进而回退至使用HTTP/2或HTTP/1。虽然这一功能仍在alpha测试阶段，需要经过生产环境的充
分验证，但它已经准备好用于生产环境。

### Happy Eyeballs（快乐眼球算法）支持

Envoy支持[RFC8305](https://tools.ietf.org/html/rfc8305)定义的Happy Eyeballs算法，用以优化上游TCP连接。对于使用[LOGICAL_DNS](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-enum-value-config-cluster-v3-cluster-discoverytype-logical-dns)
的集群，可通过将[config.cluster.v3.Cluster.DnsLookupFamily](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-enum-config-cluster-v3-cluster-dnslookupfamily)中的DNS IP地址解析策略设置为ALL，
以启用该功能并返回IPv4和IPv6地址。而对于使用[EDS](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-enum-value-config-cluster-v3-cluster-discoverytype-eds)的集群，则可通过[additional_addresses](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/endpoint/v3/endpoint_components.proto#envoy-v3-api-field-config-endpoint-v3-endpoint-additional-addresses)字段
为主机指定额外的IP地址来实现该功能。这些额外指定的地址将追加到[address](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/endpoint/v3/endpoint_components.proto#envoy-v3-api-field-config-endpoint-v3-endpoint-address)字段中指定的地址列表。

所有地址列表会根据 Happy Eyeballs 规范进行排序。首先尝试连接列表中的第一个地址，
如果连接成功，将使用该连接。如果连接失败，则尝试列表中的下一地址。如果300ms后仍未建立连接，
系统将尝试备份连接到列表中的下一个地址。

最终，如果某个地址的连接尝试成功，则此时将使用该连接；如果所有尝试都失败，则系统将报告一个连
接错误。

### 连接池数量

每个集群中的每个主机都会配置有一个或多个连接池。如果集群仅配置了一种协议，那么主机可能仅需一
个连接池。但是，如果集群支持多种上游协议，并且没有采用ALPN，那么会为每种协议分配一个连接池。
此外，以下特性也各自需要一个独立的连接池：

- [路由优先级](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/http/http_routing#arch-overview-http-routing-priority)
- [套接字选项](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/address.proto#envoy-v3-api-field-config-core-v3-bindconfig-socket-options)
- [传输套接字选项](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-transportsocket)（例如TLS）
- 那些被标记为与上游连接共享且可哈希的下游[过滤器状态对象](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/advanced/data_sharing_between_filters#arch-overview-advanced-filter-state-sharing)。

每个工作线程都会为它所服务的每个集群维护自己的连接池。因此，如果一个Envoy实例运行了两个线程，
并且有一个集群同时支持HTTP/1和HTTP/2，那么至少会存在4个连接池。

### 健康检查交互

如果Envoy启用了主动或被动健康检查功能，当主机由可用变为不可用时，所有的连接池连接都将关闭。
若该主机重新被纳入负载均衡轮询，将创建新连接，这样做将最大化规避潜在流量问题的机会，无论这些
问题是由ECMP路由还是其他因素造成的。

## 负载均衡

### 概述

#### 什么是负载均衡？

负载均衡是指在一个上游集群中，将流量在多个主机之间分配的方法，目的是为了高效利用可用资源。有
多种方法可以实现负载均衡，Envoy因此提供了多种负载均衡策略。从宏观上看，这些策略可以分为两
大类：全局负载均衡和分布式负载均衡。

#### 分布式负载均衡

分布式负载均衡指的是Envoy根据上游主机的位置决定如何分配负载到端点。

#### 示例

- [主动健康检查](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/health_checking#arch-overview-health-checking)：Envoy通过对上游主机进行健康检查，调整优先级和地区权重以适应不可用的主机。

- [区域感知路由](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/load_balancing/zone_aware#arch-overview-load-balancing-zone-aware-routing)：这种方法能使Envoy优先选择距离较近的端点，无需在控制平面中显式设置优先级。

- [负载均衡算法](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/load_balancing/load_balancers#arch-overview-load-balancing-types)：Envoy可以使用多种算法来利用所提供的权重决定选择哪一个主机。

#### 全局负载均衡

全局负载均衡是指由一个全局决策机构来决定如何在主机间分配负载。对Envoy而言，这由控制平面来进
行，它通过指定优先级、地区权重、端点权重和端点健康等参数来调整对各个端点的负载应用。

一个简单的例子是控制平面根据网络拓扑把主机分配到不同优先级，以确保网络跳数更少的主机得到优先。
这与区域感知路由相似，但是由控制平面而非Envoy执行。控制平面中实施的一个优势是它能够解决区域感
知路由的某些限制。

更高级的设置可能涉及向控制平面报告资源使用情况，以便它调整端点或地区的权重，反映当前资源使用状
况，尽可能将新请求路由至空闲主机而非繁忙主机。

#### 分布式与全局

复杂部署通常结合使用这两种特性。全局负载均衡用于定义高层次的路由优先级和权重，而分布式负载均
衡则用于实时响应系统变化，如使用主动健康检查。将这两者结合，可以实现最佳效果：一个具有全局视
角的机构能在宏观层面控制流量，同时各个代理能够在微观层面迅速适应变化。
