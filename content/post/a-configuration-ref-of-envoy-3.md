---
title: "Envoy 配置指南(三)"
date: 2024-11-30T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网配置指南的中文翻译"
Tags: ["envoy", "配置指南", "中文文档"]
Categories: ["envoy"]
DisableComments: false
---

本文档基于 v1.28, 原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/configuration

Envoy 官网配置指南的中文翻译(监听):网络过滤器
<!--more-->

除了 
[HTTP 连接管理器](https://itsfinn.cc/post/a-configuration-ref-of-envoy-2)
（其规模足够大，在配置指南中拥有自己的部分）之外，Envoy 还有以下内置网络过滤器。

# 客户端 TLS 身份验证


- 客户端 TLS 身份验证过滤器[架构概述](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/security/ssl#arch-overview-ssl-auth-filter)
- 此过滤器 URL 类型应配置为`type.googleapis.com/envoy.extensions.filters.network.client_ssl_auth.v3.ClientSSLAuth`。
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/client_ssl_auth/v3/client_ssl_auth.proto#envoy-v3-api-msg-extensions-filters-network-client-ssl-auth-v3-clientsslauth)

## 统计数据

每个配置的客户端 TLS 身份验证过滤器都有以 *auth.clientssl.\<stat_prefix\>.* 为根的统计信息，其中包含以下统计信息：

|名称|类型|描述|
|---|----|---------|
|update_success |计数器 |安全主体更新成功总数|
|update_failure |计数器 |安全主体更新失败总数|
|auth_no_ssl |计数器 |由于没有 TLS 而忽略的总连接数|
|auth_ip_allowlist |计数器 |由于 IP 允许列表而允许的总连接数|
|auth_digest_match |计数器 |由于证书匹配而允许的总连接数|
|auth_digest_no_match |计数器 |由于没有证书匹配而拒绝的总连接数|
|total_principals |计量表 |总加载安全主体数|

## REST API

> **GET /v1/certs/list/approved**

> 身份验证过滤器将在每个刷新间隔内调用此 API 来获取当前列表
> 已批准的证书/主体。预期的 JSON 响应如下所示：
>
>``` json
> {
>   "certificates":[]
> }
>```
>
> **certificates**
>
> > *（必需，数组）* 已批准的证书/主体列表。
>
> 每个证书对象定义如下：
>
>``` json
> {
> "fingerprint_sha256": "...",
> }
>```
>
> **fingerprint_sha256**
>
> > *(必填，字符串)* 已批准的客户端证书的 SHA256 哈希值。Envoy 将匹配此
> > 对所呈现的客户端证书进行哈希处理，以确定是否存在摘要匹配。

# 连接限制过滤器

- 连接限制[架构概述](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/listeners/listener_filters#arch-overview-connection-limit)
- 此过滤器 URL 类型应配置为`type.googleapis.com/envoy.extensions.filters.network.connection_limit.v3.ConnectionLimit`。
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/connection_limit/v3/connection_limit.proto#envoy-v3-api-msg-extensions-filters-network-connection-limit-v3-connectionlimit)

## 概述

过滤器可以保护连接、CPU、内存等资源，方法是确保每个过滤器链都能公平分配连接资源，并防止任何基于过滤器链匹配或描述符的单个实体消耗大量连接。连接限制过滤器将连接限制应用于由过滤器的过滤器链处理的传入连接。过滤器处理的每个连接都标记为活动连接，如果活动连接数达到最大连接限制，则将关闭该连接，而无需进一步进行过滤器迭代。

- 连接限制过滤器类似于 L4 本地速率限制过滤器，但该过滤器不是强制限制连接速率，而是限制活动连接的数量。
- 过滤器维护一个活动连接数的原子计数器。它具有基于配置的总连接数的最大连接限制值。当有新的连接请求时，过滤器会尝试增加连接计数器。如果计数器小于最大连接限制，则允许连接，否则将拒绝连接。当活动连接关闭时，过滤器会减少活动连接计数器。
- 过滤器不会停止连接创建，但会关闭已接受但被视为超出限制的连接。
- **缓慢拒绝**：过滤器可以停止读取连接并在延迟后关闭它，而不是立即拒绝它或在拒绝之前让请求通过。这样，我们可以防止恶意实体在耗尽其资源的同时打开新连接。

> **注意**
> 在当前实现中，每个过滤器链都有独立的连接限制。

## 统计数据

每个配置的连接限制过滤器都有以 *connection_limit.\<stat_prefix\>.* 为根的统计信息，其中包含以下统计信息：

|名称 |类型 |描述|
|--------------------|-------|---------------------------|
|limited_connections |计数器 |由于超出连接限制而被拒绝的连接总数|
|active_connections |计量表|此网络过滤链范围内当前活动连接的数量|

# Direct response

直接响应过滤器是一个简单的网络过滤器，用于通过可选的预设响应立即响应新的下游连接。例如，它可以用作过滤器链中的终端过滤器来收集阻塞流量的遥测数据。

- 此过滤器应配置为 URL 类型 `type.googleapis.com/envoy.extensions.filters.network.direct_response.v3.Config`
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/direct_response/v3/config.proto#envoy-v3-api-msg-extensions-filters-network-direct-response-v3-config)

# Dubbo Proxy

dubbo代理过滤器对dubbo客户端与服务端之间的RPC协议进行解码，解码后的RPC信息转换为元数据，元数据包括基本的请求ID，请求类型，序列化类型以及路由所需要的服务名，方法名，参数名，参数值。

- 此过滤器应配置为 URL 类型`type.googleapis.com/envoy.extensions.filters.network.dubbo_proxy.v3.DubboProxy`。
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/dubbo_proxy/v3/dubbo_proxy.proto#envoy-v3-api-msg-extensions-filters-network-dubbo-proxy-v3-dubboproxy)

## 统计信息

每个配置的 dubbo 代理过滤器都有以 *dubbo.\<stat_prefix\>.* 为根的统计信息，其中包含以下统计信息：

|名称 |类型 |描述|
|----------------------------------------|-------- |--------------------------------------------------------|
|request|Counter|总请求数|
|request_twoway |Counter |双向请求总数|
|request_oneway |Counter |单向请求总数|
|request_event |Counter |事件请求总数|
|request_decoding_error |Counter |总解码错误请求数|
|request_decoding_success |Counter |解码成功请求总数|
|request_active |Gauge |总活跃请求数|
|response |Counter |总回应数|
|response_success |Counter |成功响应总数|
|response_error |Counter |协议解析错误的响应总数|
|response_error_caused_connection_close |Counter |由下游连接关闭引起的响应总数|
|response_business_exception |Counter |业务层返回的协议包含异常信息的响应总数|
|response_decoding_error |Counter |总解码错误响应|
|response_decoding_success |Counter |解码成功响应总数|
|response_error |Counter |协议解析错误的响应总数|
|local_response_success |Counter |本地响应总数|
|local_response_error |Counter |编码错误的本地响应总数|
|local_response_business_exception |Counter |协议包含业务异常的本地响应总数|
|cx_destroy_local_with_active_rq |Counter|通过活动查询本地破坏的连接|
|cx_destroy_remote_with_active_rq |Counter|通过活动查询远程破坏的连接|

## 基于dubbo代理过滤器实现自定义过滤器

如果要基于dubbo协议实现自定义的filter，dubbo类似HTTP的代理filter也提供了很方便的扩展方式，第一步就是实现DecoderFilter接口，并给filter命名，比如testFilter，第二步就是添加你的配置，配置方法参考下面的示例

```yaml
filter_chains:
- filters:
  - name: envoy.filters.network.dubbo_proxy
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.network.dubbo_proxy.v3.DubboProxy
      stat_prefix: dubbo_incomming_stats
      protocol_type: Dubbo
      serialization_type: Hessian2
      multiple_route_config:
        name: local_route
        route_config:
        - interface: org.apache.dubbo.demo.DemoService
          routes:
          - match:
              method:
                name:
                  exact: sayHello
            route:
              cluster: user_service_dubbo_server
      dubbo_filters:
      - name: envoy.filters.dubbo.testFilter
        typed_config:
          "@type": type.googleapis.com/google.protobuf.Struct
          value:
            name: test_service
      - name: envoy.filters.dubbo.router
```

# Echo

echo 是一个简单的网络过滤器，主要用于演示网络过滤器 API。如果安装，它将回显（写入）所有接收到的数据给连接的下游客户端。

- 此过滤器应配置为 URL 类型 `type.googleapis.com/envoy.extensions.filters.network.echo.v3.Echo` 。
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/echo/v3/echo.proto#envoy-v3-api-msg-extensions-filters-network-echo-v3-echo)


# External Authorization


- 外部授权[架构概述](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/security/ext_authz_filter#arch-overview-ext-authz)
- 此过滤器应配置为 URL 类型`type.googleapis.com/envoy.extensions.filters.network.ext_authz.v3.ExtAuthz`。
- [网络过滤器 v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/ext_authz/v3/ext_authz.proto#envoy-v3-api-msg-extensions-filters-network-ext-authz-v3-extauthz)

外部授权网络过滤器调用外部授权服务来检查传入请求是否被授权。如果网络过滤器认为该请求未经授权，则连接将被关闭。

> **提示**
> 建议在过滤器链中首先配置此过滤器，以便在其余过滤器处理请求之前授权请求。

传递给授权服务的请求内容由 [CheckRequest](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/service/auth/v3/external_auth.proto#envoy-v3-api-msg-service-auth-v3-checkrequest) 指定。

网络过滤器 gRPC 服务可以按如下方式配置。您可以在 [网络过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/ext_authz/v3/ext_authz.proto#envoy-v3-api-msg-extensions-filters-network-ext-authz-v3-extauthz) 中查看所有配置选项。

## 例子

一个示例过滤器配置如下：

``` yaml
filters:
  - name: envoy.filters.network.ext_authz
    typed_config:
      "@type": type.googleapis.com/envoy.extensions.filters.network.ext_authz.v3.ExtAuthz
      stat_prefix: ext_authz
      grpc_service:
        envoy_grpc:
          cluster_name: ext-authz
      include_peer_certificate: true

clusters:
  - name: ext-authz
    type: static
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    load_assignment:
      cluster_name: ext-authz
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1
                port_value: 10003
```

对指定身份验证服务的示例请求主体如下所示

```json
{
  "source":{
    "address":{
      "socket_address":{
        "address": "172.17.0.1",
        "port_value": 56746
      }
    }
  }
  "destination":{
    "service": "www.bing.com",
    "address":{
      "socket_address": {
        "address": "127.0.0.1",
        "port_value": 10003
      }
    }
  }
}
```

## 统计数据

网络过滤器在 *config.ext_authz.* 命名空间中输出统计信息。

|Name                  |Type              |描述|
|----------------------|----------------- |-----------|
|total                 |Counter           |来自过滤器的总响应数。|
|error                 |Counter           |联系外部服务的错误总数。|
|denied                |Counter           |授权服务拒绝流量的响应总数。|
|disabled              |Counter           |由于过滤器被禁用而允许不调用外部服务的总请求数。|
|failure_mode_allowed  |Counter           |由于failure_mode_allow设置为true而允许通过的错误请求总数。|
|ok                    |Counter           |授权服务允许流量的响应总数。|
|cx_closed             |Counter           |关闭的连接总数。|
|active                |Gauge             |当前正在传输到授权服务的活动请求总数。|

## 动态元数据

仅当 gRPC 授权服务器返回具有非空 
[dynamic\_metadata](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/service/auth/v3/external_auth.proto#envoy-v3-api-field-service-auth-v3-checkresponse-dynamic-metadata) 
字段的 
[CheckResponse](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/service/auth/v3/external_auth.proto#envoy-v3-api-msg-service-auth-v3-checkresponse) 
时，外部授权过滤器才会将动态元数据作为不透明的 `google.protobuf.Struct` 发出。

# 通用代理


网络过滤器可用于为 Envoy 添加多种协议支持。社区已经实现了很多此类网络过滤器，例如 
[Dubbo proxy](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/listeners/network_filters/dubbo_proxy_filter#config-network-filters-dubbo-proxy)
、 
[Thrift proxy](https://www.envoyproxy.io/docs/envoy/v1.28.7/configuration/listeners/network_filters/thrift_proxy_filter#config-network-filters-thrift-proxy)
等。开发人员也可以通过创建新的网络过滤器来实现自己的协议代理。

添加新的网络过滤器以支持新协议并不容易。开发人员需要实现编解码器来解析二进制文件，实现流管理来处理请求/响应，实现连接管理来处理连接生命周期等。有许多常用功能，如路由、L7 过滤器链、跟踪、指标、日志记录等，需要在每个新的网络过滤器中反复实现。

许多 RPC 协议都有类似的基于请求/响应的架构。请求可以根据请求属性路由到不同的上游。对于这些类似的协议，它们的网络过滤器也类似，可以抽象为以下模块：

- 编解码器：将二进制文件解析为请求/响应对象。
- 流管理：处理请求/响应流。
- Route：根据请求属性将请求路由到不同的上游。
- L7 过滤器链：过滤请求/响应。
- 可观察性：跟踪、指标、日志记录等。

除了编解码器之外，其他模块都是通用的，可以被不同的协议共享。这就是通用代理的动机。

通用代理是一种网络过滤器，可用于实现新协议代理。提供了一个扩展点，让用户配置特定的编解码器。开发人员只需实现编解码器来解析二进制文件，然后让用户在通用代理中配置编解码器。通用代理将处理其余工作。

## 抽象请求/响应

请求/响应的抽象是通用代理的核心。不同的L7协议可能具有不同的请求/响应数据结构。在通用代理中，编解码器是扩展的，可以由用户配置，但其他模块是通用的，可以由不同的协议共享。这些不同协议的不同请求/响应数据结构需要抽象并在一个通用的抽象中进行管理。

定义一个抽象的 `Request` 类来表示请求。`Request` 提供了一些虚拟方法来获取/设置请求属性。不同的协议可以扩展 `Request` 并在自己的数据结构中实现虚拟方法来获取/设置请求属性。定义一个抽象类 `Response` 来表示响应。它与 `Request` 类似。

基于请求/响应的抽象，通用代理可以处理请求/响应流，将请求路由到不同的上游，过滤请求/响应等，而无需了解请求/响应的L7应用程序和具体的数据结构。

如果开发者想要实现一个新的协议代理，只需要实现编解码器来将二进制数据解析为具体的请求/响应，并确保它们实现了`Request`或`Response`。这比实现新的网络过滤器要容易得多。

## 可扩展的匹配器和路由

[通用匹配器 API](https://www.envoyproxy.io/docs/envoy/v1.28.7/intro/arch_overview/advanced/matching/matching_api#arch-overview-matching-api) 用于构建通用代理的路由表。开发人员可以扩展输入和匹配器以支持新的匹配逻辑。

默认情况下，通用代理支持以下输入和匹配器：

- [host](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/generic_proxy/matcher/v3/matcher.proto#envoy-v3-api-msg-extensions-filters-network-generic-proxy-matcher-v3-hostmatchinput)：匹配请求的主机。主机应该代表可用于负载平衡的一组实例。它可以是目标服务的 DNS 名称或 VIP。
- [path](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/generic_proxy/matcher/v3/matcher.proto#envoy-v3-api-msg-extensions-filters-network-generic-proxy-matcher-v3-pathmatchinput)：匹配请求的路径。该路径应该表示用于表示目标服务提供的一组方法或功能的 RPC 服务名称。
- [method](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/generic_proxy/matcher/v3/matcher.proto#envoy-v3-api-msg-extensions-filters-network-generic-proxy-matcher-v3-methodmatchinput)：匹配请求的方法。
- [property](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/generic_proxy/matcher/v3/matcher.proto#envoy-v3-api-msg-extensions-filters-network-generic-proxy-matcher-v3-propertymatchinput)：匹配请求的属性。该属性可以是任何通过字符串键索引的请求属性。
- [request input](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/generic_proxy/matcher/v3/matcher.proto#envoy-v3-api-msg-extensions-filters-network-generic-proxy-matcher-v3-requestmatchinput) 和 [request matcher](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/generic_proxy/matcher/v3/matcher.proto#envoy-v3-api-msg-extensions-filters-network-generic-proxy-matcher-v3-requestmatcher)：通过以 AND 语义组合主机、路径、方法和属性来匹配整个请求。这用于匹配下游请求的多个字段，并避免通用匹配树中主机、路径、方法和属性的复杂组合。

> **注意**
> 虚拟方法用于获取 `Request` 中的请求属性，如 `host()`、`path()`、`method()` 等。开发人员需要实现这些虚拟方法，并确定他们自己的协议中的这些字段是什么。协议可能没有这些字段中的某些字段。例如，协议没有 `host` 字段。在这种情况下，开发人员可以在虚拟方法中返回空字符串，以指示协议中不存在该字段。唯一的缺点是通用代理无法通过此字段匹配请求。

```yaml
              "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.codecs.dubbo.v3.DubboCodecConfig
          route_config:
            name: route_config
            virtual_hosts:
            - name: route_config_default_virtual_host
              hosts:
              - "org.apache.dubbo.UserProvider"
              routes:
                matcher_list:
                  matchers:
                  - predicate:
                      single_predicate:
                        input:
                          name: request
                          typed_config:
                            "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.matcher.v3.RequestMatchInput
                        custom_match:
                          name: request
                          typed_config:
                            "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.matcher.v3.RequestMatcher
                            host:
                              exact: "org.apache.dubbo.UserProvider"
                            method:
                              exact: "getUser"
                            properties:
                            - name: "id"
                              string_match:
                                exact: "1"
                    on_match:
                      action:
                        name: route
                        typed_config:
                          "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.action.v3.RouteAction
```

## 异步编解码器 API

通用代理提供了一个扩展点，让开发人员可以实现特定于其自身协议的编解码器。编解码器 API 设计为异步的，以避免阻塞工作线程。异步编解码器 API 可以通过将解析工作卸载到特定硬件来加速编解码器。

## 可配置连接

不同的协议可能具有不同的连接生命周期或连接管理。通用代理提供了额外的选项，让编解码器开发人员可以配置连接生命周期和连接管理。

例如，开发人员可以配置上游连接是否与下游连接绑定。如果上游连接与下游连接绑定，则上游连接将具有与下游连接相同的生命周期。绑定的上游连接将仅由来自相关下游连接的请求使用。这对于需要保持连接状态的协议很有用。

开发者还可以直接在编解码器中操作下行连接和上行连接。这让开发者对连接有了更多的控制权。

## 编解码器实现示例

社区已经实现了一个基于通用代理的 [dubbo codec](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/generic_proxy/codecs/dubbo/v3/dubbo.proto#envoy-v3-api-msg-extensions-filters-network-generic-proxy-codecs-dubbo-v3-dubbocodecconfig)。dubbo codec 复杂度适中，是一个很好的例子，可以展示如何为新的协议实现新的 codec。

您可以在 [contrib/generic\_proxy/filters/network/source/codecs/dubbo](https://github.com/envoyproxy/envoy/blob/v1.28.7/contrib/generic_proxy/filters/network/source/codecs/dubbo) 目录中找到 dubbo 编解码器实现。您还可以使用以下配置在通用代理中配置 dubbo 编解码器：

```yaml
static_resources:
  listeners:
  - name: main
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 9090
    filter_chains:
    - filters:
      - name: generic_proxy
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.v3.GenericProxy
          stat_prefix: stats_prefix
          filters:
          - name: envoy.filters.generic.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.router.v3.Router
          codec_config:
            name: dubbo
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.codecs.dubbo.v3.DubboCodecConfig
          route_config:
            name: route_config
            virtual_hosts:
            - name: route_config_default_virtual_host
              hosts:
              - "org.apache.dubbo.UserProvider"
              routes:
                matcher_list:
                  matchers:
                  - predicate:
                      single_predicate:
                        input:
                          name: request
                          typed_config:
                            "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.matcher.v3.RequestMatchInput
                        custom_match:
                          name: request
                          typed_config:
                            "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.matcher.v3.RequestMatcher
                            host:
                              exact: "org.apache.dubbo.UserProvider"
                            method:
                              exact: "getUser"
                            properties:
                            - name: "id"
                              string_match:
                                exact: "1"
                    on_match:
                      action:
                        name: route
                        typed_config:
                          "@type": type.googleapis.com/envoy.extensions.filters.network.generic_proxy.action.v3.RouteAction
                          cluster: dubbo
  clusters:
  - name: dubbo
    connect_timeout: 5s
    type: STRICT_DNS
    load_assignment:
      cluster_name: dubbo
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: localhost
                port_value: 8080
```

# Golang

Golang 网络过滤器允许 [Golang] 在下游和上游 tcp 数据流期间运行，并使扩展 Envoy 变得更加容易。

该过滤器使用的 Go 插件可以独立于 Envoy 重新编译。

有关过滤器实现的更多详细信息，请参阅[Envoy 的 Golang 扩展提案文档]。

> **警告**
> Envoy Golang 过滤器设计为在设置 `GODEBUG=cgocheck=0` 环境变量的情况下运行。
>
> 这将禁用 cgo 指针检查。
>
> 未能设置此环境变量将导致 Envoy 崩溃！

## 开发一个 Go 插件

Envoy 的 Go 插件必须实现 [DownstreamFilter/UpstreamFilter API](https://github.com/envoyproxy/envoy/blob/v1.28.7/contrib/golang/common/go/api/filter.go)。

### 构建 Go 插件

> **注意**
> 构建 Go 插件动态库时，**必须**使用与 Envoy 的 glibc 版本一致的 Go 版本。

确保兼容 Go 版本的一种方法是使用 Envoy 的 bazel 设置提供的 Go 二进制文件：

```console
$ bazel run @go_sdk//:bin/go -- version
go version goX.YZ linux/amd64
```

例如，要为 `foo` 插件构建 `.so`，您可以运行：

```console
$ bazel run @go_sdk//:bin/go build --buildmode=c-shared  -v -o path/to/output/libfoo.so path/to/src/foo
```

## 配置

> **提示**
> 此过滤器应使用类型 URL `type.googleapis.com/envoy.extensions.filters.http.golang.v3alpha.Config` 进行配置。

预先构建的 Golang 网络过滤器`my_plugin.so`可能配置如下：

```yaml
      - name: envoy.filters.network.golang
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.golang.v3alpha.Config
          is_terminal_filter: true
          library_id: simple
          library_path: "/lib/simple.so"
          plugin_name: simple
          plugin_config:
            "@type": type.googleapis.com/xds.type.v3.TypedStruct
            value:
              echo_server_addr: echo_service
```

# Kafka Broker 过滤器

Apache Kafka 代理过滤器会解码 [Apache Kafka](https://kafka.apache.org/) 的客户端协议，包括有效负载中的请求和响应。支持 [Kafka 3.5.1](http://kafka.apache.org/35/protocol.html#protocol_api_keys) 中的消息版本（ConsumerGroupHeartbeat 除外）。该过滤器会尝试不影响客户端和代理之间的通信，因此无法解码的消息（由于 Kafka 客户端或代理运行的版本比此过滤器支持的版本要新）会按原样转发。

- 此过滤器应配置为 URL 类型`type.googleapis.com/envoy.extensions.filters.network.kafka_broker.v3.KafkaBroker`。
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.28.7/api-v3/extensions/filters/network/kafka_broker/v3/kafka_broker.proto#envoy-v3-api-msg-extensions-filters-network-kafka-broker-v3-kafkabroker)

> **注意**
> Kafka 代理过滤器仅包含在 [contrib 镜像](https://www.envoyproxy.io/docs/envoy/v1.28.7/start/install#install-contrib) 中

> **注意**
> kafka_broker 过滤器处于实验阶段，目前正在积极开发中。随着时间的推移，功能将不断扩展，配置结构也可能会发生变化。

## 配置

Kafka Broker 过滤器应该与 TCP 代理过滤器链接在一起，如下面的配置片段所示：

```yaml
listeners:
- address:
    socket_address:
      address: 127.0.0.1 # Host that Kafka clients should connect to.
      port_value: 19092  # Port that Kafka clients should connect to.
  filter_chains:
  - filters:
    - name: envoy.filters.network.kafka_broker
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.kafka_broker.v3.KafkaBroker
        stat_prefix: exampleprefix
    - name: envoy.filters.network.tcp_proxy
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.tcp_proxy.v3.TcpProxy
        stat_prefix: tcp
        cluster: localkafka
clusters:
- name: localkafka
  connect_timeout: 0.25s
  type: strict_dns
  lb_policy: round_robin
  load_assignment:
    cluster_name: some_service
    endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 127.0.0.1 # Kafka broker's host
                port_value: 9092 # Kafka broker's port.
```

Kafka 代理需要宣告 Envoy 监听器端口而不是自己的端口。

```text
# Listener value needs to be equal to cluster value in Envoy config
# (will receive payloads from Envoy).
listeners=PLAINTEXT://127.0.0.1:9092

# Advertised listener value needs to be equal to Envoy's listener
# (will make clients discovering this broker talk to it through Envoy).
advertised.listeners=PLAINTEXT://127.0.0.1:19092
```

## 统计数据

每个配置的 Kafka Broker 过滤器的统计数据都以 *kafka.\<stat_prefix\>.* 为根，并且有多个
每种消息类型的统计信息。

|Name                    |Type             |描述|
|------------------------|-----------------|--------------------------------------------------------|
|request.TYPE            |Counter          |从 Kafka 客户端收到特定类型请求的次数|
|request.unknown         |Counter          |收到此过滤器无法识别格式的请求的次数|
|request.failure         |Counter          |收到格式无效的请求或发生其他处理异常的次数|
|response.TYPE           |Counter          |从 Kafka 代理收到特定类型响应的次数|
|response.TYPE_duration  |Histogram        |响应生成时间（以毫秒为单位）|
|response.unknown        |Counter          |收到此过滤器无法识别格式的响应的次数|
|response.failure        |Counter          |收到格式无效的响应或发生其他处理异常的次数|

# Kafka 网格过滤器


Apache Kafka 网格过滤器为 
[Apache Kafka](https://kafka.apache.org/)
集群封装了一个接口层。

它允许处理下游客户端发送的 Produce（生产者）和 Fetch（消费者）请求。

该过滤器实例收到的请求可以转发到多个集群之一，具体取决于配置的转发规则。

支持从Kafka 3.5.1开始的相应消息版本。

- 此过滤器应配置为 URL 类型`type.googleapis.com/envoy.extensions.filters.network.kafka_mesh.v3alpha.KafkaMesh`。
- [v3 API 参考](https://www.envoyproxy.io/docs/envoy/v1.26.8/api-v3/extensions/filters/network/kafka_mesh/v3alpha/kafka_mesh.proto#envoy-v3-api-msg-extensions-filters-network-kafka-mesh-v3alpha-kafkamesh)

> **注意**
> Kafka 网格过滤器仅包含在 [contrib 镜像](https://www.envoyproxy.io/docs/envoy/v1.26.8/start/install#install-contrib) 中

> **注意**
> kafka_mesh 过滤器处于实验阶段，目前正在积极开发中。随着时间的推移，其功能将不断扩展，配置结构也可能会发生变化。

> **注意**
> kafka_mesh 过滤器在 Windows 上不起作用（阻止程序正在编译 librdkafka）。

## 配置

下面的示例向我们展示了代理 3 个 Kafka 集群的典型过滤器配置。客户端将连接到 \'127.0.0.1:19092\'，并且它们的消息将根据主题名称分发到集群。

``` yaml
listeners:
- address:
    socket_address:
      address: 127.0.0.1 # Host that Kafka clients should connect to.
      port_value: 19092  # Port that Kafka clients should connect to.
  filter_chains:
  - filters:
    - name: envoy.filters.network.kafka_mesh
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.filters.network.kafka_mesh.v3alpha.KafkaMesh
        advertised_host: "127.0.0.1"
        advertised_port: 19092
        upstream_clusters:
        - cluster_name: kafka_c1
          bootstrap_servers: cluster1_node1:9092,cluster1_node2:9092,cluster1_node3:9092
          partition_count: 1
        - cluster_name: kafka_c2
          bootstrap_servers: cluster2_node1:9092,cluster2_node2:9092,cluster2_node3:9092
          partition_count: 1
        - cluster_name: kafka_c3
          bootstrap_servers: cluster3_node1:9092,cluster3_node2:9092
          partition_count: 5
          producer_config:
            acks: "1"
            linger.ms: "500"
          consumer_config:
            client.id: "my-envoy-consumer"
        forwarding_rules:
        - target_cluster: kafka_c1
          topic_prefix: apples
        - target_cluster: kafka_c2
          topic_prefix: bananas
        - target_cluster: kafka_c3
          topic_prefix: cherries
```

值得注意的是，可以在过滤器链中的 Kafka 代理过滤器之前插入 Kafka 网格过滤器，以捕获请求处理指标。

## 注释

1. 使用嵌入式 [librdkafka](https://github.com/confluentinc/librdkafka) 生产者/消费者发送/接收记录。
2. librdkafka 编译时没有使用 ssl、lz4、gssapi，因此不支持相关的自定义配置选项。
3. 启动时未发现无效的自定义配置（仅在初始化适当的生产者或消费者时）。引用这些集群的请求将关闭连接并失败。
4. 引用与任何规则都不匹配的主题的请求将关闭连接并失败。这通常不应该发生（客户端首先请求元数据，然后它们应该首先失败并出现“无可用代理”），但如果有人通过连接定制二进制有效负载，则可能发生这种情况。

## 生产者代理

1. 指向上游 Kafka 集群的嵌入式 librdkafka 生产者是每个 Envoy 工作线程创建的（因此可以使用 *--concurrency* 选项增加吞吐量，从而允许更多生产者处理请求）。
2. 仅支持版本 2 的 ProduceRequests（这意味着像 0.8 这样的非常旧的生产者将不受支持）。
3. Python 生产者需要设置至少 1.0.0 的 API 版本，以便他们发送的生产请求将具有 magic 等于 2 的记录。
4. Kafka 生产器 'acks' 属性的下游处理委托给上游客户端。例如，如果上游客户端配置为使用 acks=0，则响应将尽快发送给下游客户端（即使它们有非零的 acks！）。
5. 由于过滤器将单个生产者请求拆分为单独的记录，因此可能只有部分记录的交付失败。在这种情况下，返回给上游客户端的响应是失败的，但是部分记录可能已附加到目标集群中。
6. 由于上述拆分，记录不一定是一条接一条地附加的（因为它们不会作为单个请求发送到上游）。想要避免这种情况的用户可能需要查看下游生产者配置：'linger.ms' 和 'batch.size'。

## 消费者代理

1. 目前，消费者代理仅支持有状态代理 - Envoy 使用指向上游的 librdkafka 消费者来接收记录，并且仅在请求更多数据时才这样做。
2. 用户可能需要查看消费者的配置属性*group.id*来管理消费者的偏移提交行为（在 Envoy 重启时有意义的）。
3. 当请求消费者位置时，响应始终包含 offset = 0（参见*list_offsets.cc*）。
4. 提供了记录偏移信息，但没有提供记录批量偏移增量——据观察，Apache Kafka Java 客户端尽管接收到记录，但不会更新其位置（参见*fetch_record_converter.cc*）。
5. 如果收集到至少 3 条记录，则 Fetch 响应将发送到下游（请参阅 *fetch.cc*）。当前实现将忽略请求中有关请求字节等的数据。
6. 在被视为已完成（见上文）或 5 秒的硬编码超时（见 *fetch.cc*）过后，将发送 Fetch 响应。请求指定的超时将被忽略。
7. 只要有针对这些主题的传入请求，消费者就会轮询这些主题的记录，而不考虑分区。鼓励用户确保所有分区都被使用，以避免出现以下情况：例如，我们只从分区 0 获取记录，但代理会收到分区 0（发送到下游）和分区 1（保存在内存中，直到有人对它们感兴趣）的记录。
