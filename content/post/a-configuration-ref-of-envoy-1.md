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

此外，还支持以下统计数据来表明由于缺少配置而导致连接被关闭，
其根是 listener.<address>（或 listener.<stat_prefix>。如果 stat_prefix 非空）。


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
[cds.proto](https://github.com/envoyproxy/envoy/blob/79958991ffe0cf8d59a4a351646c76d672dada83/api/envoy/service/cluster/v3/cds.proto)
了解服务定义。当 Envoy 以此作为客户端时，

```yaml
eds_config:
  api_config_source:
    api_type: GRPC
    grpc_services:
    - envoy_grpc:
        cluster_name: some_xds_cluster
```
