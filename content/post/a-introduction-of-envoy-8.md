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

# 请求流

## 概述

使用上述配置示例简要概述请求和响应的生命周期：

1. 由工作线程上的Envoy监听器接受来自下游的TCP连接。
2. 创建并运行监听器过滤器链。它可以提供SNI和其他TLS前的信息。
   一旦完成，监听器将匹配一个网络过滤器链。
   每个监听器可以有多个过滤器链，它们根据目标IP CIDR范围、SNI、ALPN、源端口等的组合进行匹配。
   在这种情况下，传输套接字（TLS传输套接字）与该过滤器链关联。
3. 在网络读取时，TLS传输套接字解密从TCP连接读取的数据到解密的数据流以供进一步处理。
4. 创建并运行网络过滤器链。对于HTTP来说，最重要的过滤器是HTTP连接管理器，它是链中的最后一个网络过滤器。
5. HTTP连接管理器中的HTTP/2编解码器将来自TLS连接的解密数据流拆分为多个独立流。每个流处理单个请求和响应。
6. 对于每个HTTP流，创建并运行下游HTTP过滤器链。
   请求首先通过CustomFilter，该过滤器可能读取和修改请求。
   最重要的HTTP过滤器是位于HTTP过滤器链末端的路由器过滤器。
   当对路由器过滤器调用decodeHeaders时，选择路由并选择一个集群。
   该集群中的请求头将转发到上游端点。
   路由器过滤器从集群管理器中获取与匹配集群相关的HTTP连接池进行此操作。
7. 执行集群特定的负载均衡以找到端点。
   检查集群的断路器以确定是否允许新的流。
   如果端点的连接池为空或容量不足，则创建与该端点的新连接。
8. 对于每个流，创建并运行上游HTTP过滤器链。
   默认情况下，这仅包括发送数据到适当编解码器的CodecFilter，
   但如果集群配置了上游过滤器链，则将为每个流创建和运行该过滤器链，
   包括为重试和阴影请求创建和运行单独的过滤器链。
9. 上游端点连接的HTTP/2编解码器通过单个TCP连接将请求流的任何其他流与该上游进行多路复用和帧化。
10. 上游端点连接的TLS传输套接字加密这些字节，并将它们写入上游连接的TCP套接字。
11. 请求（包括头部、可选正文和尾部）被代理到上游，响应被代理到下游。
    响应通过HTTP过滤器的顺序与请求相反，从编解码器过滤器开始，遍历任何上游过滤器，
    然后通过路由器过滤器并经过CustomFilter，最后发送到下游。
12. 当响应完成时，流将被销毁。后请求处理将更新统计信息、写入访问日志并完成跟踪跨度。

我们将在以下部分详细阐述这些步骤。

## 1. 监听器 TCP Accept

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-listeners.svg)

监听器管理器负责接收表示监听器的配置，并实例化一些绑定到各自IP/端口的Listener实例。监听器可以处于以下三种状态之一：

- 预热（Warming）：监听器正在等待配置依赖项（例如路由配置、动态秘钥）。监听器尚未准备好接受TCP连接。

- 活动（Active）：监听器绑定到其IP/端口并接受TCP连接。

- 排空（Draining）：监听器不再接受新的TCP连接，但其现有的TCP连接在一段时间内仍被允许继续。

每个工作线程为其配置的每个监听器维护自己的Listener实例。
每个监听器可以通过 SO_REUSEPORT 或共享单个绑定到此端口的套接字来绑定到相同的端口。
当新的TCP连接到达时，内核决定哪个工作线程将接受该连接，并且该工作线程的Listener的`Server::ConnectionHandlerImpl::ActiveTcpListener::onAccept()`回调将被调用。

## 2. 监听器过滤器链和网络过滤器链匹配

工作线程的Listener然后创建并运行监听器过滤器链。过滤器链是通过应用每个过滤器的过滤器工厂创建的。工厂知道过滤器的配置，并为每个连接或流创建过滤器的新实例。

对于我们的TLS监听器配置，监听器过滤器链由[TLS检查器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/listener_filters/tls_inspector#config-listener-filters-tls-inspector)过滤器（envoy.filters.listener.tls_inspector）组成。此过滤器检查初始TLS握手并提取服务器名称（SNI）。然后，SNI可用于过滤器链匹配。尽管TLS检查器在监听器过滤器链配置中显式出现，但Envoy还能够在需要SNI（或ALPN）时自动插入此过滤器。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-listener-filters.svg)

TLS检查器过滤器实现了 [ListenerFilter](https://github.com/envoyproxy/envoy/blob/v1.28.0/envoy/network/filter.h)接口。
所有过滤器接口，无论是监听器还是网络/HTTP，都要求过滤器为特定的连接或流事件实现回调。对于 `ListenerFilter`，这是：

```
virtual FilterStatus onAccept(ListenerFilterCallbacks& cb) PURE;
```

`onAccept()` 允许过滤器在 TCP accept 处理期间运行。回调返回的`FilterStatus`控制监听器过滤器链将如何继续。监听器过滤器可以暂停过滤器链，然后稍后恢复，例如响应于对另一个服务的RPC。

从监听器过滤器和连接属性中提取的信息随后用于匹配过滤器链，提供将用于处理连接的网络过滤器链和传输套接字。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-filter-chain-match.svg)

3. TLS传输套接字解密

Envoy通过[TransportSocket](https://github.com/envoyproxy/envoy/blob/v1.28.0/envoy/network/transport_socket.h)扩展接口提供可插拔的传输套接字。
传输套接字遵循TCP连接的生命周期事件，并读取/写入网络缓冲区。传输套接字必须实现的一些关键方法如下：

```
virtual void onConnected() PURE;
virtual IoResult doRead(Buffer::Instance& buffer) PURE;
virtual IoResult doWrite(Buffer::Instance& buffer, bool end_stream) PURE;
virtual void closeSocket(Network::ConnectionEvent event) PURE;
```

当TCP连接上有数据可用时，`Network::ConnectionImpl::onReadReady()``通过`SslSocket::doRead()``调用TLS传输套接字。
传输套接字然后在TCP连接上执行TLS握手。
当握手完成后，`SslSocket::doRead()``向负责管理网络过滤器链的`Network::FilterManagerImpl``实例提供解密的字节流。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-transport-socket.svg)

重要的是要注意，没有任何操作，无论是TLS握手还是过滤器管道的暂停，都是真正的阻塞操作。
由于Envoy是基于事件的，任何需要额外数据才能完成处理的情况都会导致早期事件完成，并将CPU让给其他事件。
当网络提供更多可读数据时，一个读取事件将触发TLS握手的恢复。

## 4. 网络过滤器链处理

与监听器过滤器链一样，Envoy通过 `Network::FilterManagerImpl` 实例化一系列网络过滤器，从它们的过滤器工厂中。
每个新连接都有一个新的实例。
网络过滤器，就像传输套接字一样，遵循TCP生命周期事件，并在传输套接字可用数据时被调用。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-network-filters.svg)

网络过滤器以管道的形式组成，与每个连接一个的传输套接字不同。网络过滤器有三种类型：

- [ReadFilter](https://github.com/envoyproxy/envoy/blob/v1.28.0/envoy/network/filter.h) 实现 `onData()`，当从连接中可用数据时调用（由于某些请求）。

- [WriteFilter](https://github.com/envoyproxy/envoy/blob/v1.28.0/envoy/network/filter.h) 实现 `onWrite()`，当数据即将写入连接时调用（由于某些响应）。

- 实现ReadFilter和WriteFilter的[Filter](https://github.com/envoyproxy/envoy/blob/v1.28.0/envoy/network/filter.h)。

关键过滤器方法的方法签名如下：

```
virtual FilterStatus onNewConnection() PURE;
virtual FilterStatus onData(Buffer::Instance& data, bool end_stream) PURE;
virtual FilterStatus onWrite(Buffer::Instance& data, bool end_stream) PURE;
```

与监听器过滤器一样，`FilterStatus`允许过滤器暂停过滤器链的执行。例如，如果需要查询限速服务，限速网络过滤器将返回`Network::FilterStatus::StopIteration`从`onData()`，并在查询完成后调用`continueReading()`。

处理HTTP的监听器的最后一个网络过滤器是HTTP连接管理器（HCM）。它负责创建HTTP/2编解码器和管理HTTP过滤器链。在我们的示例中，这是唯一的网络过滤器。使用多个网络过滤器的示例网络过滤器链如下所示：

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-network-read.svg)

在响应路径上，网络过滤器链按与请求路径相反的顺序执行。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-network-write.svg)

## 5. HTTP/2 codec 解码

Envoy中的HTTP/2编解码器基于nghttp2。它由HCM使用来自TCP连接的明文字节（经过网络过滤器链转换后）调用。编解码器将字节流解码为一系列HTTP/2帧，并将连接解复用为多个独立的HTTP流。流复用是HTTP/2的关键功能，与HTTP/1相比提供了显著的性能优势。每个HTTP流处理一个请求和响应。

编解码器还负责处理HTTP/2设置帧以及流和连接级别的[流量控制](https://github.com/envoyproxy/envoy/blob/v1.28.0/source/docs/flow_control.md)。

编解码器负责抽象HTTP连接的细节，向HTTP连接管理器和HTTP过滤器链呈现标准的连接视图，该连接分为多个流，每个流具有请求/响应头/正文/尾部。无论协议是HTTP/1、HTTP/2还是HTTP/3，这都是正确的。

## 6. HTTP过滤器链处理

对于每个HTTP流，HCM实例化一个[下游HTTP过滤器链](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/http/http_filters#arch-overview-http-filters)，
遵循上述监听器和网络过滤器链的模式。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/lor-http-filters.svg)

有三种HTTP过滤器接口：

- [StreamDecoderFilter](https://github.com/envoyproxy/envoy/blob/v1.28.0/envoy/http/filter.h)与请求处理回调。

- [StreamEncoderFilter](https://github.com/envoyproxy/envoy/blob/v1.28.0/envoy/http/filter.h)与响应处理回调。

- [StreamFilter](https://github.com/envoyproxy/envoy/blob/v1.28.0/envoy/http/filter.h) 实现 `StreamDecoderFilter` 和`StreamEncoderFilter`。

查看解码器过滤器接口：

```
virtual FilterHeadersStatus decodeHeaders(RequestHeaderMap& headers, bool end_stream) PURE;
virtual FilterDataStatus decodeData(Buffer::Instance& data, bool end_stream) PURE;
virtual FilterTrailersStatus decodeTrailers(RequestTrailerMap& trailers) PURE;
```
```
virtual FilterHeadersStatus decodeHeaders(RequestHeaderMap& headers, bool end_stream) PURE;
virtual FilterDataStatus decodeData(Buffer::Instance& data, bool end_stream) PURE;
virtual FilterTrailersStatus decodeTrailers(RequestTrailerMap& trailers) PURE;
```
