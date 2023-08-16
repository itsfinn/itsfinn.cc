---
title: "HTTPRoute - Kubernetes Gateway API"
date: 2023-08-16T12:00:00+08:00
isCJKLanguage: true
Description: "HTTPRoute - Kubernetes Gateway API"
Tags: ["k8s", "kubernetes", "HTTPRoute"]
Categories: ["k8s", "kubernetes"]
DisableComments: false
---


# HTTPRoute

`HTTPRoute` 是一种 `Gateway API` 类型，用于指定从 `Gateway` 侦听器到 API 对象（即`Service`）的 HTTP 请求的路由行为。

## Spec

HTTPRoute 的规范包括：

- `ParentRefs` - 定义此路由想要附加到哪些网关。
- `Hostnames`（可选）- 定义用于匹配 HTTP 请求的主机标头的主机名列表。
- `Rules` - 定义规则列表以针对匹配的 HTTP 请求执行操作。每个规则由`matches`、`filters`（可选）和 `backendRefs`（可选）字段组成。

下图说明了将所有流量发送到一个服务的 HTTPRoute

![httproute-basic-example](https://gateway-api.sigs.k8s.io/images/httproute-basic-example.svg)

### 关联到网关

每个路由都包含一种引用它想要附加到的父资源的方法。在大多数情况下，这将是 `Gateway`，但这里的实现有一定的灵活性来支持其他类型的父资源。

以下示例显示路由如何附加到 `acme-lb` 网关：

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: httproute-example
spec:
  parentRefs:
  - name: acme-lb
```

请注意，目标网关需要允许附加来自路由命名空间的 `HTTPRoute` 才能成功附加。

### 主机名(Hostnames)

主机名定义与 HTTP 请求的主机标头相匹配的主机名列表。当匹配发生时，根据规则和过滤器（可选）选择 HTTPRoute 执行请求路由。

主机名是网络主机的完全限定域名，如 [RFC 3986](https://tools.ietf.org/html/rfc3986) 所定义。请注意与 RFC 中定义的 URI 的 "host" 部分的以下偏差：

- 不允许使用 IP。
- 由于不允许使用端口，因此不考虑":"分隔符。

在评估 HTTPRoute 规则之前，传入请求将与主机名进行匹配。如果未指定主机名，则根据 HTTPRoute 规则和过滤器（可选）路由流量。

以下示例定义主机名 "my.example.com"：

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: httproute-example
spec:
  hostnames:
  - my.example.com

```

### 规则

规则定义用于根据条件匹配 HTTP 请求、可选地执行附加处理步骤以及可选地将请求转发到 API 对象的语义。

#### Matches

匹配定义用于匹配 HTTP 请求的条件。每个匹配都是独立的，即如果满足任何单个匹配，则将匹配该规则。

以下面的匹配配置为例：

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
...
spec:
  rules:
  - matches:
    - path:
        value: "/foo"
      headers:
      - name: "version"
        value: "2"
    - path:
        value: "/v2/foo"
```
对于匹配此规则的请求，它必须满足以下条件之一：

- 以 /foo 为前缀 **且** 包含标头 "version: 2" 的路径
- /v2/foo 的路径前缀

如果没有配置 `matches` ，则默认是 "/" 上的前缀路径匹配，其效果是匹配每个HTTP请求。

#### Filters (optional)

过滤器定义在请求或响应生命周期中必须完成的处理步骤。过滤器充当扩展点来表达可以在网关实现中执行的附加处理。

一些示例包括请求或响应修改、实施身份验证策略、速率限制和流量整形。

以下示例将标头 "my-header: foo" 添加到主机标头 "my.filter.com" 的 HTTP 请求。

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: http-filter-1
spec:
  hostnames:
    - my.filter.com
  rules:
    - filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: my-header
                value: foo
      backendRefs:
        - name: my-filter-svc1
          weight: 1
          port: 80
```

API 一致性是根据过滤器类型定义的。目前尚未明确对多种行为进行排序的效果。根据 alpha 阶段的反馈，这可能会在未来发生变化。

一致性级别由过滤器类型定义：

- 所有 "core" 过滤器必须得到实现的支持。
- 鼓励实施者支持 "extended" 过滤器。
- "Implementation-specific" 过滤器没有跨实现的 API 保证。

多次指定 core 过滤器具有未指定或特定于实现的一致性。

除了 URLRewrite 和 RequestRedirect 过滤器之外，所有过滤器都应相互兼容，这两个过滤器可能无法组合。如果实现不能支持其他过滤器组合，则必须在文档中描述清楚对应的限制。

如果指定了不兼容或不受支持的过滤器并导致 Accepted 条件设置为状态 False ，实现可以使用 `IncompatibleFilters` 原因来指定此配置错误。

#### BackendRefs（可选）
BackendRefs defines API objects where matching requests should be sent. If unspecified, the rule performs no forwarding. If unspecified and no filters are specified that would result in a response being sent, a 404 error code is returned.
BackendRefs 定义匹配的请求应该发送到的 API 对象。如果未指定，则该规则不执行转发。如果未指定且未指定会导致发送响应的过滤器，则返回 404 错误代码。

以下示例将前缀 /bar 的 HTTP 请求转发到 "my-service1" 服务的 8080 端口上，并将前缀为 /some/thing 并且带 `magic: foo`标头的 HTTP 请求转发到 "my-service2" 服务的 8080 端口上：

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: acme-lb
spec:
  controllerName: acme.io/gateway-controller
  parametersRef:
    name: acme-lb
    group: acme.io
    kind: Parameters
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: acme-lb
  listeners:  # Use GatewayClass defaults for listener definition.
  - name: http
    protocol: HTTP
    port: 80
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: http-app-1
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "foo.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /bar
    backendRefs:
    - name: my-service1
      port: 8080
  - matches:
    - headers:
      - type: Exact
        name: magic
        value: foo
      queryParams:
      - type: Exact
        name: great
        value: example
      path:
        type: PathPrefix
        value: /some/thing
      method: GET
    backendRefs:
    - name: my-service2
      port: 8080
```

以下示例使用 weight 字段将 foo.example.com 的 HTTP 请求的 90% 转发到"foo-v1"服务，将另外 10% 转发到"foo-v2"服务：

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: foo-route
  labels:
    gateway: prod-web-gw
spec:
  hostnames:
  - foo.example.com
  rules:
  - backendRefs:
    - name: foo-v1
      port: 8080
      weight: 90
    - name: foo-v2
      port: 8080
      weight: 10
```

有关 weight 和其他字段的更多详细信息，请参阅 backendRef API 文档。

## Status

Status 定义了 HTTPRoute 的观察状态。

### RouteStatus

RouteStatus 定义所有路由类型所需的观察状态。

### Parents

父级定义与 HTTPRoute 关联的网关（或其他父级资源）的列表，以及与每个网关相关的 HTTPRoute 的状态。

当 HTTPRoute 在parentRefs中添加对网关的引用时，管理网关的控制器应在控制器第一次看到该路由时向此列表添加一个条目，并应在修改路由时适当更新该条目。

以下示例表明 HTTPRoute "http-example" 已被名称空间 "gw-example-ns" 中的网关 "gw-example" 接受：

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: http-example
...
status:
  parents:
  - parentRefs:
      name: gw-example
      namespace: gw-example-ns
    conditions:
    - type: Accepted
      status: "True"
```

## Merging

多个 HTTPRoutes 可以附加到单个网关资源。重要的是，每个请求只能匹配一个路由规则。有关冲突解决如何应用于合并的更多信息，请参阅 API 规范。
