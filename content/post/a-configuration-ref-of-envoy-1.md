---
title: "Envoy 配置指南(一)"
date: 2024-11-18T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网配置指南的中文翻译"
Tags: ["envoy", "配置指南", "中文文档"]
Categories: ["envoy"]
DisableComments: false
---

本文档基于 v1.28, 原文地址: [https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/configuration)

Envoy 官网配置指南的中文翻译(概览)
<!--more-->

# 介绍

Envoy xDS API 在 [api 树](https://github.com/envoyproxy/envoy/blob/v1.28.7/api/)
中定义为
[proto3 协议缓冲区](https://developers.google.com/protocol-buffers/)。
它们支持：

- 通过 gRPC流式传输 [xDS](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-docs/xds_protocol#xds-protocol)
  API 更新。这减少了资源需求并可以降低更新延迟。

- 一种新的 REST-JSON API，其中 JSON/YAML 格式是通过 [proto3 规范 JSON 映射](https://developers.google.com/protocol-buffers/docs/proto3#json)
  机械派生的。

- 通过文件系统、REST-JSON 或 gRPC 端点传递更新。

- 通过扩展的端点分配 API 实现高级负载平衡，并向管理服务器报告负载和资源利用率。

- 在需要时提供[更强的一致性和排序属性](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-docs/xds_protocol#xds-protocol-eventual-consistency-considerations)。
  API 仍然维护基线最终一致性模型。

有关 Envoy 和管理服务器之间 xDS 消息交换的更多详细信息，请参阅 [xDS 协议描述](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-docs/xds_protocol#xds-protocol)。

# 版本控制

Envoy xDS API 遵循明确定义的[版本控制方案](https://github.com/envoyproxy/envoy/blob/v1.28.7/api/API_VERSIONING.md)。

Envoy 具有针对 xDS 传输（即用于在管理服务器和 Envoy 之间移动资源的有线协议）和资源的 API 版本。这些分别称为传输和资源 API 版本。

传输和资源 API 版本均遵循 API 版本支持和弃用[政策](https://github.com/envoyproxy/envoy/blob/v1.28.7/api/API_VERSIONING.md)。

# 引导配置

要使用 xDS API，必须提供引导配置文件。这提供了静态服务器配置，并配置 Envoy 以
[在需要时访问动态配置](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/operations/dynamic_configuration#arch-overview-dynamic-config)。
这通过命令行上的标志提供 `-c`，即：

```/shell
./envoy -c <path to config>.{json,yaml,pb,pb_text}
```

其中文件名扩展反映了底层配置表示格式。

Bootstrap消息是配置的根源。Bootstrap 消息中的一个关键概念是静态资源和动态资源之间的区别。
Listener或Cluster 等资源可以在static_resources中静态提供 ，也可以在dynamic_resources中配置 xDS 服务（例如LDS或CDS ）。

# 示例

下面我们将使用配置协议的 YAML 表示和一个从 127.0.0.1:10000 到 127.0.0.1:1234 代理 HTTP 的服务运行示例。

## 静态配置

下面提供了最小的完全静态引导配置：

```yaml
admin:
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 127.0.0.1, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: some_service }
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
  - name: some_service
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: some_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 1234
```

## 大部分为静态，采用动态 EDS

下面提供了一个引导配置，该配置继续上面的示例，通过 监听 127.0.0.1:5678 的 [EDS gRPC 管理服务器进行动态端点发现](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/operations/dynamic_configuration#arch-overview-dynamic-config-eds)：

```shell
admin:
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 127.0.0.1, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: some_service }
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
  - name: some_service
    connect_timeout: 0.25s
    lb_policy: ROUND_ROBIN
    type: EDS
    eds_cluster_config:
      eds_config:
        resource_api_version: V3
        api_config_source:
          api_type: GRPC
          transport_api_version: V3
          grpc_services:
            - envoy_grpc:
                cluster_name: xds_cluster
  - name: xds_cluster
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options:
            connection_keepalive:
              interval: 30s
              timeout: 5s
    upstream_connection_options:
      # configure a TCP keep-alive to detect and reconnect to the admin
      # server in the event of a TCP socket half open connection
      tcp_keepalive: {}
    load_assignment:
      cluster_name: xds_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 5678
```

请注意，上面定义了xds_cluster以将 Envoy 指向管理服务器。
即使在完全动态的配置中，也需要定义一些静态资源以将 Envoy 指向其 xDS 管理服务器。

在块中设置适当 [的TCP Keep-Alive 选项](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/core/v3/address.proto#envoy-v3-api-msg-config-core-v3-tcpkeepalive)
`tcp_keepalive` 很重要。这将有助于检测与 xDS 管理服务器的 TCP 半开连接并重新建立完整连接。

在上面的例子中，EDS 管理服务器可以返回 [DiscoveryResponse](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/service/discovery/v3/discovery.proto#envoy-v3-api-msg-service-discovery-v3-discoveryresponse)
的原始编码

```yaml
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment
  cluster_name: some_service
  endpoints:
  - lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: 127.0.0.1
            port_value: 1234
```

上面出现的版本控制和类型 URL 方案在 [流式 gRPC 订阅协议](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-docs/xds_protocol#xds-protocol-streaming-grpc-subscriptions)
文档中有更详细的解释 。

## 动态配置

下面提供了一个完全动态的引导配置，其中除属于管理服务器的资源之外的所有资源都通过 xDS 发现：

```yaml
admin:
  address:
    socket_address: { address: 127.0.0.1, port_value: 9901 }

dynamic_resources:
  lds_config:
    resource_api_version: V3
    api_config_source:
      api_type: GRPC
      transport_api_version: V3
      grpc_services:
        - envoy_grpc:
            cluster_name: xds_cluster
  cds_config:
    resource_api_version: V3
    api_config_source:
      api_type: GRPC
      transport_api_version: V3
      grpc_services:
        - envoy_grpc:
            cluster_name: xds_cluster

static_resources:
  clusters:
  - name: xds_cluster
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options:
            # Configure an HTTP/2 keep-alive to detect connection issues and reconnect
            # to the admin server if the connection is no longer responsive.
            connection_keepalive:
              interval: 30s
              timeout: 5s
    load_assignment:
      cluster_name: xds_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 5678
```

管理服务器可以使用以下方式响应 LDS 请求：

```yaml
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.config.listener.v3.Listener
  name: listener_0
  address:
    socket_address:
      address: 127.0.0.1
      port_value: 10000
  filter_chains:
  - filters:
    - name: envoy.filters.network.http_connection_manager
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
        stat_prefix: ingress_http
        codec_type: AUTO
        rds:
          route_config_name: local_route
          config_source:
            resource_api_version: V3
            api_config_source:
              api_type: GRPC
              transport_api_version: V3
              grpc_services:
                - envoy_grpc:
                    cluster_name: xds_cluster
        http_filters:
        - name: envoy.filters.http.router
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
```

管理服务器可以使用以下方式响应 RDS 请求：

```yaml
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.config.route.v3.RouteConfiguration
  name: local_route
  virtual_hosts:
  - name: local_service
    domains: ["*"]
    routes:
    - match: { prefix: "/" }
      route: { cluster: some_service }
```

管理服务器可以使用以下方式响应 CDS 请求:

```yaml
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.config.cluster.v3.Cluster
  name: some_service
  connect_timeout: 0.25s
  lb_policy: ROUND_ROBIN
  type: EDS
  eds_cluster_config:
    eds_config:
      resource_api_version: V3
      api_config_source:
        api_type: GRPC
        transport_api_version: V3
        grpc_services:
          - envoy_grpc:
              cluster_name: xds_cluster
```

管理服务器可以使用以下方式响应 EDS 请求：

```yaml
version_info: "0"
resources:
- "@type": type.googleapis.com/envoy.config.endpoint.v3.ClusterLoadAssignment
  cluster_name: some_service
  endpoints:
  - lb_endpoints:
    - endpoint:
        address:
          socket_address:
            address: 127.0.0.1
            port_value: 1234
```

## 特殊 YAML 用法

加载 YAML 配置时，Envoy 加载器将特殊解释带有 !ignore 标记的映射键，并将其从本机配置树中完全忽略。
通常，YAML 流必须严格遵守为 Envoy 配置定义的 proto 架构。这允许声明明确处理为非表示类型的内容。

这样您就可以将文件拆分为两部分：一部分是不需要根据架构进行解析的 YAML 内容，另一部分是需要解析的 YAML 内容。第一部分中的 YAML 锚点可以由第二部分中的别名引用。此机制可以简化需要重复使用或动态生成配置片段的设置。

请参阅以下示例：

```yaml
!ignore dynamic_sockets:
- &admin_address {address: 127.0.0.1, port_value: 9901}
- &listener_address {address: 127.0.0.1, port_value: 10000}
- &lb_address {address: 127.0.0.1, port_value: 1234}

admin:
  address:
    socket_address: *admin_address

static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: *listener_address
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: {prefix: "/"}
                route: {cluster: some_service}
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
  - name: some_service
    connect_timeout: 0.25s
    type: STATIC
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: some_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: *lb_address
```

> **警告**
> 如果您使用外部加载器解析 Envoy YAML 配置，则可能需要将 !ignore 标签告知这些加载器。
> 兼容的 YAML 加载器通常会公开一个接口，让您选择如何处理自定义标签。

例如，这将指示 [PyYAML](https://github.com/yaml/pyyaml) 在加载时将忽略的节点视为简单标量：

```shell
yaml.SafeLoader.add_constructor('!ignore', yaml.loader.SafeConstructor.construct_scalar)
```
或者，这是 Envoy 在配置验证中注册 !ignore 标签的[方式](https://github.com/envoyproxy/envoy/blob/v1.28.7/tools/config_validation/validate_fragment.py)。

# 扩展配置

Envoy 中的每个配置资源在 中都有一个类型 URL `typed_config`。
此类型对应于版本化架构。类型 URL 唯一地标识能够解释配置的扩展。
该name字段是可选的，可以用作标识符或扩展配置特定实例的注释。例如，允许以下过滤器配置代码段：

```yaml
      - name: front-http-proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          rds:
            route_config_name: local_route
            config_source:
              resource_api_version: V3
              api_config_source:
                api_type: GRPC
                transport_api_version: V3
                grpc_services:
                - envoy_grpc:
                    cluster_name: xds_cluster
          http_filters:
          - name: front-router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
              dynamic_stats: true
```

如果控制平面缺少扩展的架构定义， 应该使用 `xds.type.v3.TypedStruct` 作为通用容器。
然后，客户端会使用其中的类型 URL 将内容转换为类型化配置资源。例如，上述示例可以写成如下形式：

```yaml
      - name: front-http-proxy
        typed_config:
          "@type": type.googleapis.com/xds.type.v3.TypedStruct
          type_url: type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          value:
            stat_prefix: ingress_http
            codec_type: AUTO
            rds:
              route_config_name: local_route
              config_source:
                resource_api_version: V3
                api_config_source:
                  api_type: GRPC
                  transport_api_version: V3
                  grpc_services:
                  - envoy_grpc:
                      cluster_name: xds_cluster
            http_filters:
            - name: front-router
              typed_config:
                "@type": type.googleapis.com/xds.type.v3.TypedStruct
                type_url: type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
                value:
                  dynamic_stats: true
```

## 发现服务

可以使用 [ExtensionConfiguration 发现服务](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/service/extension/v3/config_discovery.proto#envoy-v3-api-file-envoy-service-extension-v3-config-discovery-proto)
从
[xDS 管理服务器](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-docs/xds_protocol#xds-protocol)
动态提供扩展配置。扩展配置中的名称字段充当资源标识符。

## 监听器过滤器

对于侦听器过滤器，发现服务配置为：[动态侦听器过滤器重新配置](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-field-config-listener-v3-listenerfilter-config-discovery)。
动态侦听器过滤器配置仅在 TCP 侦听器中受支持。如果缺少动态配置，则将拒绝连接，直到更新有效的配置。

## 网络过滤器

对于下游网络过滤器，发现服务配置为：[动态过滤器重新配置](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-field-config-listener-v3-filter-config-discovery)。
如果缺少动态配置，则连接将被拒绝，直到更新有效的配置。
当过滤器配置更新时，新配置将仅适用于新连接，现有连接将继续使用旧的过滤器配置。

## HTTP 过滤器

对于 HTTP 过滤器，HTTP 连接管理器支持[动态过滤器重新配置](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpfilter-config-discovery)。
如果配置缺失，将返回带有“500”状态代码的本地 HTTP 响应。

## 统计数据

除了xDS 订阅支持的统计信息之外，还支持侦听器过滤器、下游网络过滤器和 HTTP 过滤器的以下统计信息，其根为 extension_config_discovery.<stat_prefix>.<extension_config_name>。

- 对于TCP侦听器过滤器， <stat_prefix> 的值为 *tcp_listener_filter* 。
- 对于下游网络过滤器，<stat_prefix> 的值为 *network_filter*。
- 对于上游网络过滤器，<stat_prefix> 的值为 *upper_network_filter*。
- 对于下游 HTTP 过滤器，<stat_prefix> 的值为 *http_filter*。
- 对于上游 HTTP 过滤器，<stat_prefix> 的值为 *upper_http_filter*。

|名称|类型|描述|
|---|---|---|
|config_reload|Counter|成功配置更新的总数|
|config_fail|Counter|配置更新失败总数|
|config_conflict|Counter|配置更新冲突的应用程序总数；当新的侦听器由于类型 URL 无效而无法重用已订阅的扩展配置时，可能会发生这种情况。|

此外，还支持以下统计数据来表明由于缺少配置而导致连接被关闭, 其根是 listener.\<address\>（或 listener.<stat_prefix>。如果 stat_prefix 非空）。


|名称|类型|描述|
|---|---|---|
|extension_config_missing|Counter|由于缺少侦听器过滤器扩展配置，关闭的连接总数|
|network_extension_config_missing|Counter|由于缺少网络过滤器扩展配置而关闭的连接总数|

# xDS API 端点

xDS 管理服务器将根据 gRPC 和(或) REST 服务的要求实现以下端点。在流式 gRPC 和 REST-JSON 情况下，都会按照
[xDS 协议](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#xds-protocol)
发送
[DiscoveryRequest](https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/discovery/v3/discovery.proto#envoy-v3-api-msg-service-discovery-v3-discoveryrequest)
并接收
[DiscoveryResponse](https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/discovery/v3/discovery.proto#envoy-v3-api-msg-service-discovery-v3-discoveryresponse)。

下面我们描述 v3 传输 API 的端点。

## gRPC 流式传输端点

> **POST /envoy.service.cluster.v3.ClusterDiscoveryService/StreamClusters**

请参阅
[cds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/cluster/v3/cds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
    api_config_source:
      api_type: GRPC
      grpc_services:
      - envoy_grpc:
          cluster_name: xds_cluster
  lds_config:
    api_config_source:
      api_type: GRPC
```

在
[Bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-msg-config-bootstrap-v3-bootstrap)
配置的 
[dynamic_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-field-config-bootstrap-v3-bootstrap-dynamic-resources)
中设置。

> **POST /envoy.service.endpoint.v3.EndpointDiscoveryService/StreamEndpoints**

请参阅 
[eds.proto](https://github.com/envoyproxy/envoy/blob/v1.28.7/api/envoy/service/endpoint/v3/eds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
eds_config:
  api_config_source:
    api_type: GRPC
    grpc_services:
    - envoy_grpc:
        cluster_name: some_xds_cluster
```

在 
[Cluster](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-msg-config-cluster-v3-cluster)
配置的
[eds_cluster_config](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-eds-cluster-config)
字段中设置。

> **POST /envoy.service.listener.v3.ListenerDiscoveryService/StreamListeners**

请参阅
[lds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/listener/v3/lds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
  lds_config:
    resource_api_version: V3
    api_config_source:
      api_type: GRPC
      transport_api_version: V3
      grpc_services:
      - envoy_grpc:
          cluster_name: xds_cluster
```

在 [Bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-msg-config-bootstrap-v3-bootstrap)
配置的
[dynamic_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-field-config-bootstrap-v3-bootstrap-dynamic-resources)
中设置。

> **POST /envoy.service.route.v3.RouteDiscoveryService/StreamRoutes**

请参阅 
[rds.proto](https://github.com/envoyproxy/envoy/blob/v1.28.7/api/envoy/service/route/v3/rds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
route_config_name: some_route_name
config_source:
  api_config_source:
    api_type: GRPC
    grpc_services:
    - envoy_grpc:
        cluster_name: some_xds_cluster
```

在
[HttpConnectionManager](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-msg-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager)
配置的
[rds](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-rds)
字段中设置。

> **POST /envoy.service.route.v3.ScopedRoutesDiscoveryService/StreamScopedRoutes**

请参阅 
[srds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/route/v3/srds.proto)了解服务定义。当 Envoy 以此作为客户端时，

```yaml
name: some_scoped_route_name
scoped_rds:
  config_source:
    api_config_source:
      api_type: GRPC
      grpc_services:
      - envoy_grpc:
          cluster_name: some_xds_cluster
```
了解服务定义。

在 
[HttpConnectionManager](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-msg-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager)
配置的
[scoped_routes](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-scoped-routes)
字段中设置。

> **POST /envoy.service.secret.v3.SecretDiscoveryService/StreamSecrets**

请参阅
[sds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/secret/v3/sds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
                  token_secret:
                    name: token
                    sds_config:
                      api_config_source:
                        api_type: GRPC
                        grpc_services:
                        - envoy_grpc:
                            cluster_name: sds_server_uds
                  hmac_secret:
                    name: hmac
```

在
[SdsSecretConfig](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/transport_sockets/tls/v3/secret.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-sdssecretconfig)
消息中设置。此消息用于
[CommonTlsContext](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/transport_sockets/tls/v3/tls.proto#envoy-v3-api-msg-extensions-transport-sockets-tls-v3-commontlscontext)
等各种地方。

> **POST /envoy.service.runtime.v3.RuntimeDiscoveryService/StreamRuntime**

请参阅
[rtds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/runtime/v3/rtds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
name: some_runtime_layer_name
config_source:
  api_config_source:
    api_type: GRPC
    grpc_services:
    - envoy_grpc:
        cluster_name: some_xds_cluster
```

在[rtds_layer](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-field-config-bootstrap-v3-runtimelayer-rtds-layer)
字段内设置。

## REST 端点

> **POST /v3/discovery:clusters**

请参阅 
[cds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/cluster/v3/cds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
cds_config:
  api_config_source:
    api_type: REST
    cluster_names: [some_xds_cluster]
```

在 
[Bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-msg-config-bootstrap-v3-bootstrap)
配置的 
[dynamic_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-field-config-bootstrap-v3-bootstrap-dynamic-resources)
中设置。

> **/v3/discovery:endpoints**

请参阅
[eds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/endpoint/v3/eds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
eds_config:
  api_config_source:
    api_type: REST
    cluster_names: [some_xds_cluster]
```

在 
[Cluster](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-msg-config-cluster-v3-cluster)
配置的
[eds_cluster_config(https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-eds-cluster-config)
字段中设置。

> **POST /v3/discovery:listeners**

请参阅
[lds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/listener/v3/lds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
lds_config:
  api_config_source:
    api_type: REST
    cluster_names: [some_xds_cluster]
```

> **POST /v3/discovery:routes**

请参阅
[rds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/route/v3/rds.proto) 了解服务定义。当 Envoy 以此作为客户端时，

```yaml
route_config_name: some_route_name
config_source:
  api_config_source:
    api_type: REST
    cluster_names: [some_xds_cluster]
```

在
[HttpConnectionManager](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-msg-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager)
配置的
[rds](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-rds)
字段中设置。

> **注意**
> 响应这些端点的管理服务器必须使用 [DiscoveryResponse](https://www.envoyproxy.io/docs/envoy/latest/api-v3/service/discovery/v3/discovery.proto#envoy-v3-api-msg-service-discovery-v3-discoveryresponse) 以及 HTTP 状态 200 进行响应。此外，如果提供的配置没有改变（如 Envoy 客户端提供的版本所示），那么管理服务器可以用空主体和 HTTP 状态 304 进行响应。

## 聚合发现服务

虽然 Envoy 从根本上采用了最终一致性模型，但 ADS 提供了一个机会来对 API 更新推送进行排序，并确保单个管理服务器对 Envoy 节点的 API 更新具有亲和性。ADS 允许管理服务器在单个双向 gRPC 流上传递一个或多个 API 及其资源。如果没有此功能，某些 API（例如 RDS 和 EDS）可能需要管理多个流和与不同管理服务器的连接。

ADS 将允许通过适当的排序无中断地更新配置。例如，假设 *foo.com* 已映射到群集 *X*。我们希望更改路由表中的映射以将 *foo.com* 指向群集 *Y*。为了做到这一点，必须首先提供包含群集 *X* 和 *Y* 的 CDS/EDS 更新。

如果没有 ADS，CDS/EDS/RDS 流可能会指向不同的管理服务器，或者在同一管理服务器上指向需要协调的不同 gRPC 流/连接。EDS 资源请求可能会分为两个不同的流，一个用于 *X*，一个用于 *Y*。ADS允许将它们合并为单个流并发送到单个管理服务器，从而无需分布式同步来正确排序更新。使用 ADS，管理服务器将在单个流上提供 CDS、EDS 和 RDS 更新。

ADS 仅适用于 gRPC 流式传输（不适用于 REST），在 [xDS](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#xds-protocol-ads) 文档中有更详细的描述。gRPC 端点是：

> **POST /envoy.service.discovery.v3.AggregatedDiscoveryService/StreamAggregatedResources**

请参阅 
[discovery.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/discovery/v3/discovery.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
  ads_config:
    api_type: GRPC
    grpc_services:
    - envoy_grpc:
        cluster_name: xds_cluster
  cds_config:
```

在
[Bootstrap](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-msg-config-bootstrap-v3-bootstrap)
配置的
[dynamic_resources](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-field-config-bootstrap-v3-bootstrap-dynamic-resources)
中设置。

设置此项后，[上述](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/xds_api#v3-grpc-streaming-endpoints)任何配置源都可以设置为使用 ADS 通道。例如，LDS 配置可以从

```yaml
lds_config:
  resource_api_version: V3
  api_config_source:
    api_type: REST
    transport_api_version: V3
    cluster_names: [some_xds_cluster]
```

到

```yaml
lds_config: {ads: {}}
```

结果是 LDS 流将通过共享 ADS 通道定向到 *some_ads_cluster*。

## 增量端点

REST、文件系统和原始 gRPC xDS 实现都提供“世界状态”更新：每个 CDS 更新都必须包含每个集群，更新中缺少集群意味着集群已消失。对于拥有大量资源甚至少量流失的 Envoy 部署，这些世界状态更新可能会很麻烦。

从 1.12.0 开始，Envoy 支持 xDS（包括 ADS）的“delta”变体，其中更新仅包含添加/更改/删除的资源。Delta xDS 是一种 gRPC（唯一）协议。Delta 使用与 SotW（DeltaDiscovery{Request,Response}）不同的请求/响应协议；请参阅 discovery.proto。从概念上讲，delta 应该被视为一种新的 xDS 传输类型：有静态、文件系统、REST、gRPC-SotW 以及现在的 gRPC-delta。（Envoy 对 gRPC-SotW/delta 客户端的实现恰好在两者之间共享了大部分代码，并且在服务器端可能也会出现类似的事情。然而，它们实际上是不兼容的协议。delta xDS 协议行为的规范在这里。）

要使用 delta，只需将ApiConfigSource原型的 api_type 字段设置 为 DELTA_GRPC。这适用于 xDS 和 ADS；对于 ADS，它是 DynamicResources.ads_config的 api_type 字段，如上一节所述。

## TTL

使用 xDS 时，用户可能会发现自己想要临时更新某些 xDS 资源。为了安全地执行此操作，可以使用 xDS TTL 来确保如果控制平面不可用且无法恢复 xDS 更改，Envoy 将在服务器指定的 TTL 后删除该资源。 有关更多信息，请参阅协议文档。

目前，TTL 过期时的行为是删除资源（而不是恢复到以前的版本）。因此，此功能主要应用于希望资源不存在而不是临时版本的用例，例如使用 RTDS 应用临时运行时覆盖时。

TTL 在资源协议上指定：对于 Delta xDS，它直接在响应中指定，而对于 SotW xDS，服务器可能会将响应中列出的各个资源包装在 资源中，以指定 TTL 值。

服务器可以通过对同一版本发出另一个响应来刷新或修改 TTL。在这种情况下，不必包含资源本身。

# 管理服务器

## 管理服务器无法访问

当 Envoy 实例与管理服务器失去连接时，Envoy 将锁定以前的配置，同时在后台主动重试以重新建立与管理服务器的连接。

重要的是，Envoy 能够检测到与管理服务器的连接何时不健康，以便可以尝试建立新连接。 建议在连接到管理服务器的集群中配置 
[TCP keep-alive](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-upstreamconnectionoptions-tcp-keepalive)
或
[HTTP/2 keepalive](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-http2protocoloptions-connection-keepalive) 。

Envoy 调试记录了每次尝试连接时无法与管理服务器建立连接的事实。

[Connected_state](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/mgmt_server#management-server-stats)
统计数据提供了监控此行为的信号。

## 统计数据

管理服务器有一个以 *control_plane* 为根的统计树，其中包含以下统计数据：


|名称|类型|描述|
|---|---|---|
|connected_state|Gauge|布尔值（1 表示已连接，0 表示已断开连接），指示与管理服务器的当前连接状态|
|rate_limit_enforced|Counter|对管理服务器请求实施速率限制的总次数|
|pending_requests|Gauge|强制执行速率限制时待处理的请求总数|
|identifier|TextReadout|发送最后一个发现响应的控制平面实例的标识符|

## xDS 订阅统计

Envoy 通过称为 xDS 的发现服务发现各种动态资源。Envoy 通过订阅的方式获取这些资源，具体的方式是指定要监视的文件系统路径, 启动 gRPC 流或轮询 REST-JSON URL。

Envoy 会为所有订阅生成以下统计数据。



|名称|类型|描述|
|---|---|---|
|config_reload|Counter|由于配置不同而导致配置重新加载的 API 提取总数|
|config_reload_time_ms|Gauge|自 Unix Epoch(1970年1月1日00:00:00 UTC)以来最后一次配置重新加载的时间戳（以毫秒为单位）|
|init_fetch_timeout|Counter|初始获取超时|
|update_attempt|Counter|尝试获取 API 的总数|
|update_success|Counter|成功完成的 API 提取总数|
|update_failure|Counter|由于网络错误而失败的 API 提取总数|
|update_rejected|Counter|由于架构/验证错误而失败的 API 提取总数|
|update_time|Gauge|上次成功获取 API 的时间戳，以自Unix Epoch(1970年1月1日00:00:00 UTC)以​​来的毫秒数表示。即使在不包含任何配置更改的简单配置重新加载后也会刷新。|
|version|Gauge|上次成功 API 获取的内容的哈希值|
|version_text|TextReadout|上次成功获取 API 的版本文本|
|control_plane.connected_state|Gauge|布尔值（1 表示已连接，0 表示已断开连接），指示与管理服务器的当前连接状态|
