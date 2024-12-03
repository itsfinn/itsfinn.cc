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
