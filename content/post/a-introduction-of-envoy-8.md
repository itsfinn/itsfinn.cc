---
title: "Envoy 介绍之(八)"
date: 2024-01-13T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---

原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(请求的生命周期)
<!--more-->

以下我们将描述一个请求通过 Envoy 代理时的生命周期中的事件。
我们首先描述 Envoy 如何适应请求路径，然后描述从下游到达 Envoy 代理后发生的内部事件。
我们将跟踪请求，直到相应的上游分发和响应路径。

# 术语

Envoy的代码库和文档中用到了以下术语：

- 集群(Cluster)：一组端点，代表一个逻辑服务。Envoy会将请求转发给这些端点。
- 下游(Downstream)：与Envoy建立连接的实体。这可以是一个本地应用程序（在sidecar模型中）或网络节点。
  在非sidecar模型中，它是一个远程客户端。
- 端点(Endpoints)：实现逻辑服务的网络节点。这些节点被组合成集群。在集群中的端点是Envoy代理的上游。
- 过滤器(Filter)：处理请求的某个方面的模块，位于连接或请求处理管道中。
  它可以被视为由Unix管道连接的小工具（过滤器）的集合。
- 过滤器链(Filter chain)：一系列过滤器，它们按顺序执行。
- 监听器(Listeners)：负责绑定到IP/端口、接受新的TCP连接（或UDP数据报）并协调请求处理下游方面的Envoy模块。
- 上游：当转发服务请求时，Envoy连接到的端点（网络节点）。
  这可以是一个本地应用程序（在sidecar模型中）或网络节点。
  在非sidecar模型中，它对应于远程后端。

# 网络拓扑

请求如何在包括Envoy在内的网络组件中流动，取决于网络的拓扑结构。Envoy可以用于各种网络拓扑。
下面我们将重点关注Envoy的内部操作，但在此部分中，我们将简要介绍Envoy如何与网络的其他部分相关联。

Envoy最初是一个服务网格边车代理，将负载均衡、路由、可观察性、安全性和发现服务从应用程序中分离出来。
在服务网格模型中，请求通过Envoy作为网关流经网络。请求通过入口或出口监听器到达Envoy：

- 入口监听器从服务网格中的其他节点接收请求，并将它们转发给本地应用程序。本地应用程序的响应通过Envoy返回到下游节点。

- 出口监听器接收来自本地应用程序的请求，并将它们转发到网络中的其他节点。
  这些接收节点通常也会运行Envoy，并使用其入口监听器接收请求。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-topology-service-mesh.svg)
![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-topology-service-mesh-node.svg)

Envoy 可用于服务网格之外的各种配置。例如，它还可以充当内部负载均衡器：

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-topology-ilb.svg)

或者作为网络边缘的入口/出口代理：

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-topology-edge.svg)

在实践中，经常使用这些的混合，其中 Envoy 在服务网格、边缘和内部负载均衡器中发挥作用。
一个请求路径可能会遍历多个 Envoy。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-topology-hybrid.svg)

Envoy 可以配置为多层拓扑，以实现可扩展性和可靠性，请求首先通过边缘 Envoy，然后再通过第二个 Envoy 层：

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-topology-tiered.svg)

在上述所有情况下，请求将从下游通过 TCP、UDP 或 Unix 域套接字到达特定 Envoy。
Envoy 将通过 TCP、UDP 或 Unix 域套接字向上游转发请求。下面我们重点关注单个 Envoy 代理。

# 配置

Envoy是一个非常容易扩展的平台。这导致了可能的请求路径组合的爆炸性增长，具体取决于以下因素：

- L3/L4协议，例如TCP、UDP、Unix域套接字。
- L7协议，例如HTTP/1、HTTP/2、HTTP/3、gRPC、Thrift、Dubbo、Kafka、Redis和各种数据库。
- 传输套接字，例如明文、TLS、ALTS。
- 连接路由，例如PROXY协议、原始目的地、动态转发。
- 身份验证和授权。
- 断路器和异常检测的配置和激活状态。
- 许多其他用于网络、HTTP、监听器、访问日志记录、健康检查、跟踪和统计扩展的配置。

一次专注于一个配置很有帮助，因此此示例涵盖了以下内容：

- 下游和上游都通过TCP连接使用TLS的HTTP/2请求。
- HTTP连接管理器作为唯一的网络过滤器。
- 假设的CustomFilter和路由器过滤器作为HTTP过滤器链。
- 文件系统访问日志记录。
- Statsd接收器。
- 具有静态端点的单个集群。

为了简单起见，我们假设使用静态引导配置文件。

```yaml
static_resources:
  listeners:
  # There is a single listener bound to port 443.
  - name: listener_https
    address:
      socket_address:
        protocol: TCP
        address: 0.0.0.0
        port_value: 443
    # A single listener filter exists for TLS inspector.
    listener_filters:
    - name: "envoy.filters.listener.tls_inspector"
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.listener.tls_inspector.v3.TlsInspector
    # On the listener, there is a single filter chain that matches SNI for acme.com.
    filter_chains:
    - filter_chain_match:
        # This will match the SNI extracted by the TLS Inspector filter.
        server_names: ["acme.com"]
      # Downstream TLS configuration.
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.DownstreamTlsContext
          common_tls_context:
            tls_certificates:
            - certificate_chain: {filename: "certs/servercert.pem"}
              private_key: {filename: "certs/serverkey.pem"}
      filters:
      # The HTTP connection manager is the only network filter.
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          use_remote_address: true
          http2_protocol_options:
            max_concurrent_streams: 100
          # File system based access logging.
          access_log:
          - name: envoy.access_loggers.file
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
              path: "/var/log/envoy/access.log"
          # The route table, mapping /foo to some_service.
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["acme.com"]
              routes:
              - match:
                  path: "/foo"
                route:
                  cluster: some_service
          # CustomFilter and the HTTP router filter are the HTTP filter chain.
          http_filters:
          # - name: some.customer.filter
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
  - name: some_service
    # Upstream TLS configuration.
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
    load_assignment:
      cluster_name: some_service
      # Static endpoint assignment.
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 10.1.2.10
                port_value: 10002
        - endpoint:
            address:
              socket_address:
                address: 10.1.2.11
                port_value: 10002
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options:
            max_concurrent_streams: 100
  - name: some_statsd_sink
  # The rest of the configuration for statsd sink cluster.
# statsd sink.
stats_sinks:
- name: envoy.stat_sinks.statsd
  typed_config:
    "@type": type.googleapis.com/envoy.config.metrics.v3.StatsdSink
    tcp_cluster_name: some_statsd_sink
```

# 高级架构

Envoy中的请求处理路径有两个主要部分：

- [监听器子系统](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/listeners/listeners#arch-overview-listeners)，
  负责处理下游请求处理。它还负责管理下游请求生命周期以及客户端的响应路径。下游HTTP/2编解码器位于此处。

- [集群子系统](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/cluster_manager#arch-overview-cluster-manager)，
  负责选择和配置到端点的上游连接。这是集群和端点健康状况、负载均衡和连接池知识的所在之处。上游HTTP/2编解码器位于此处。

这两个子系统通过HTTP路由器过滤器进行桥接，将HTTP请求从下游转发到上游。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-architecture.svg)

我们使用[监听器子系统](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/listeners/listeners#arch-overview-listeners)
和[集群子系统](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/cluster_manager#arch-overview-cluster-manager)
这两个术语来指代由顶级ListenerManager和ClusterManager类创建的模块和实例类组。
我们将在下面讨论许多组件，这些组件在请求之前和请求过程中由这些管理系统实例化，
例如监听器、过滤器链、编解码器、连接池和负载均衡数据结构。

Envoy具有[基于事件的多线程模型](https://blog.envoyproxy.io/envoy-threading-model-a8d44b922310)。
主线程负责服务器生命周期、配置处理、统计信息等，而一些[工作线程](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/intro/threading_model#arch-overview-threading)处理请求。
所有线程都围绕事件循环（[libevent](https://libevent.org/)）运行，对于给定的下游TCP连接（包括其上的所有复用流），在其生命周期中，将由一个工作线程精确处理。
每个工作线程维护自己的TCP连接池到上游端点。
UDP处理使用SO_REUSEPORT，使内核始终将源/目标IP：端口元组哈希到相同的工作线程。
对于给定的工作线程，UDP过滤器状态是共享的，过滤器负责提供所需的会话语义。
这与我们下面讨论的面向连接的TCP过滤器不同，其中过滤器状态存在于每个连接和HTTP过滤器的每个请求上。

工作线程很少共享状态，并以非常简单的并行方式运行。这种线程模型使得能够扩展到非常高的核心数CPU。
