---
title: "Envoy 介绍之(二)"
date: 2023-12-18T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---


Envoy 官网介绍文档的中文翻译(上游集群部分), 原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro
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

好的，以下是按您的要求更新后的表格。

| Name | Type | Description |
| --- | --- | --- |
| resolve_total | Count | DNS查询的总数 |
| pending_resolutions | Gauge | 待处理的DNS查询的数量 |
| not_found | Counter | 返回NXDOMAIN或NODATA响应的DNS查询数量 |
| timeout | Counter | 超时的DNS查询数量 |
| get_addr_failure | Counter | DNS查询过程中发生的常规故障数量 |

基于Apple的DNS解析器在`dns.apple`统计树中发出以下统计信息：

好的，以下是按您的要求更新后的表格。

| Name | Type | Description |
| :---: | :---: | :---: |
| connection_failure | Counter | 连接到DNS服务器的失败尝试的数量 |
| get_addr_failure | Counter | 调用GetAddrInfo API时发生的一般故障数量 |
| network_failure | Counter | 由于网络连接问题导致的故障数量 |
| processing_failure | Counter | 处理来自DNS服务器数据时发生的故障数量 |
| socket_failure | Counter | 获取到DNS服务器套接字的文件描述符的失败尝试的数量 |
| timeout | Counter | 超时的查询的数量 |
