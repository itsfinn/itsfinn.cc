---
title: "GRPCRoute - Kubernetes Gateway API"
date: 2023-08-16T12:00:00+08:00
isCJKLanguage: true
Description: "GRPCRoute - Kubernetes Gateway API"
Tags: ["k8s", "kubernetes", "GRPCRoute"]
Categories: ["k8s", "kubernetes"]
DisableComments: false
---


# `GRPCRoute`

> 实验频道
> ---
> 下面描述的 `GRPCRoute` 资源当前仅包含在 Gateway API 的 "实验" 频道中。有关发布渠道的更多信息，请参阅相关文档。

`GRPCRoute` 是一种网关 API 类型，用于指定从网关侦听器到 API 对象（即服务）的 gRPC 请求的路由行为。

## 背景

虽然可以使用 HTTPRoutes 或自定义的 CRD 路由 gRPC，但从长远来看，这会导致生态系统支离破碎。

gRPC 是[业界广泛采用的流行 RPC 框架](https://grpc.io/about/#whos-using-grpc-and-why)。该协议在 Kubernetes 项目本身中广泛使用，作为许多接口的基础，包括：

- the CSI
- the CRI
- [设备插件框架](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

鉴于 gRPC 在应用层网络空间，特别是 Kubernetes 项目中的重要性，我们决定不允许生态系统出现不必要的碎片。

### 封装的网络协议

一般来说，当可以在较低层路由封装协议时，在满足以下条件时，在较高层引入路由资源是可以接受的：

- 如果被迫在较低层进行路由，封装协议的用户将错过其生态系统中重要的传统功能。
- 如果被迫在较低层进行路由，封装协议的用户将体验到降级的用户体验。
- 封装的协议拥有重要的用户群，特别是在 Kubernetes 社区。

gRPC 满足所有这些标准，因此决定在网关 API 中包含 `GRPCRoute` 。

### 交叉服务

支持 `GRPCRoute` 的实现必须强制 `GRPCRoute` 和 HTTPRoute 之间主机名的唯一性。如果要将一个 HTTPRoute 或 `GRPCRoute` 类型的路由 (A) 附加到侦听器，但是该侦听器已经附加了其他类型的另一个路由 (B)，并且主机名的交集A和B非空，那么执行必须拒绝Route A。

也就是说，实现必须在相应的 RouteParentStatus 中引发状态为 `False` 的 `Accepted` 条件。

一般来说，建议对 gRPC 和非 gRPC HTTP 流量使用单独的主机名。

这符合 gRPC 社区的标准实践。但是，如果需要在同一主机名上提供 HTTP 和 gRPC 服务，且唯一的区别是 URI，则用户应为 gRPC 和 HTTP 使用 HTTPRoute 资源。这将以 `GRPCRoute` 资源的用户体验为代价。

## Spec

`GRPCRoute` 的规范包括：

- ParentRefs - 定义此路由想要附加到哪些网关。
- Hostnames （可选）- 定义用于匹配 gRPC 请求的主机标头的主机名列表。
- Rules - 定义规则列表以针对匹配的 gRPC 请求执行操作。每个规则由 matches、filters（可选）和 backendRefs（可选）字段组成。

下图说明了将所有流量发送到一个服务的 `GRPCRoute`：

![GRPCRoute-basic-example](https://gateway-api.sigs.k8s.io/images/grpcroute-basic-example.png)

### 关联到网关

每个路由都包含一种引用它想要附加到的父资源的方法。在大多数情况下，这将是网关，但这里的实现有一定的灵活性来支持其他类型的父资源。

以下示例显示路由如何附加到 `acme-lb` 网关：
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: `GRPCRoute`
metadata:
  name: `GRPCRoute`-example
spec:
  parentRefs:
  - name: acme-lb
```
Note that the target Gateway needs to allow `GRPCRoute`s from the route's namespace to be attached for the attachment to be successful.
请注意，目标网关需要允许附加来自 `GRPCRoute` 命名空间中的 `GRPCRoute` 才能成功附加。

### 主机名(Hostnames)

主机名定义与 gRPC 请求的主机标头匹配的主机名列表。当匹配发生时，根据规则和过滤器（可选）选择 `GRPCRoute` 执行请求路由。

主机名是网络主机的完全限定域名，如 [RFC 3986](https://tools.ietf.org/html/rfc3986) 所定义。请注意与 RFC 中定义的 URI 的 "host" 部分存在以下偏差：

- 不允许使用 IP。
- 由于不允许使用端口，因此不考虑":"分隔符。

在评估 `GRPCRoute` 规则之前，传入请求将与主机名进行匹配。如果未指定主机名，则根据 `GRPCRoute` 规则和过滤器（可选）路由流量。

以下示例定义主机名 "my.example.com"：

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: `GRPCRoute`
metadata:
  name: `GRPCRoute`-example
spec:
  hostnames:
  - my.example.com
```

### 规则(Rules)

规则定义用于根据条件匹配 gRPC 请求、可选地执行附加处理步骤以及可选地将请求转发到 API 对象的语义。

#### Matches

匹配定义用于匹配 gRPC 请求的条件。每个匹配都是独立的，即如果满足任何单个匹配，则将匹配该规则。

以下面的匹配配置为例：

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: `GRPCRoute`
...
matches:
  - method:
      service: com.example.User
      method: Login
    headers:
      values:
        version: "2"
  - method:
      service: com.example.v2.User
      method: Login
```

对于匹配此规则的请求，它必须满足以下条件之一：

- `com.example.User.Login` 方法 AND 包含标头 "version: 2"
- `com.example.v2.User.Login` 方法。

如果未指定匹配项，则默认匹配每个 gRPC 请求。

#### Filters (可选的)

过滤器定义在请求或响应生命周期中必须完成的处理步骤。过滤器充当扩展点来表达可以在网关实现中执行的附加处理。

一些示例包括请求或响应修改、实施身份验证策略、速率限制和流量整形。

以下示例将标头 "my-header: foo" 添加到主机标头 "my.filter.com" 的 gRPC 请求中。请注意，`GRPCRoute` 使用 HTTPRoute 过滤器来实现与 HTTPRoute 功能相同的功能，如下所示。

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: `GRPCRoute`
metadata:
  name: grpc-filter-1
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
          port: 50051
```

API 一致性是根据过滤器类型定义的。目前尚未明确对多种行为进行排序的效果。根据 alpha 阶段的反馈，这可能会在未来发生变化。

一致性级别由过滤器类型定义：

- 所有 "core" 过滤器必须得到实现的支持。
- 鼓励实施者支持 "extended" 过滤器。
- "Implementation-specific" 过滤器没有跨实现的 API 保证。

多次指定 core 过滤器具有未指定或特定于实现的一致性。

如果实现不能支持过滤器组合，则必须清楚地记录该限制。如果指定了不兼容或不受支持的过滤器并导致 `Accepted` 条件设置为状态 `False` ，实现可以使用 `IncompatibleFilters` 原因来指定此配置错误。

#### BackendRefs（可选）

BackendRefs 定义了匹配请求应发送到的 API 对象。如果未指定，则该规则不执行转发。如果未指定且未指定会导致发送响应的过滤器，则返回 `UNIMPLEMENTED` 错误代码。

以下示例将方法 `User.Login` 的 gRPC 请求转发到 "my-service1" 服务的 50051 端口上，并将方法 `Things.DoThing` 并且带 `magic: foo` 标头的 gRPC 请求转发到 "my-service2" 服务的 8080 端口上：

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
  - name: https
    protocol: HTTPS
    port: 50051
    tls:
      certificateRefs:
      - kind: Secret
        group: ""
        name: example-com-cert
---
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: `GRPCRoute`
metadata:
  name: grpc-app-1
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "example.com"
  rules:
  - matches:
    - method:
        service: com.example.User
        method: Login
    backendRefs:
    - name: my-service1
      port: 50051
  - matches:
    - headers:
      - type: Exact
        name: magic
        value: foo
      method:
        service: com.example.Things
        method: DoThing
    backendRefs:
    - name: my-service2
      port: 50051
```

以下示例使用 weight 字段将 foo.example.com 的 gRPC 请求的 90% 转发到"foo-v1"服务，将另外 10% 转发到"foo-v2"服务：

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: `GRPCRoute`
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
      port: 50051
      weight: 90
    - name: foo-v2
      port: 50051
      weight: 10
```

有关 `weight` 和其他字段的更多详细信息，请参阅 [backendRef API]() 文档。

## Status

状态定义 `GRPCRoute` 的观察状态。

### RouteStatus

RouteStatus 定义所有路由类型所需的观察状态。

#### Parents

父级定义与 `GRPCRoute` 关联的网关（或其他父级资源）的列表，以及与每个网关相关的 `GRPCRoute` 的状态。当 `GRPCRoute` 在 parentRefs 中添加对网关的引用时，管理网关的控制器应在控制器第一次看到该路由时向此列表添加一个条目，并应在修改路由时适当更新该条目。


## 例子

以下示例表明 `GRPCRoute` "grpc-example" 已被名称空间 "grpc-example-ns" 中的网关 "grpc-example" 接受：

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: `GRPCRoute`
metadata:
  name: grpc-example
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

## 合并

多个 `GRPCRoute` 可以附加到单个网关资源。重要的是，每个请求只能匹配一个路由规则。有关冲突解决如何应用于合并的更多信息，请参阅 API 规范。
