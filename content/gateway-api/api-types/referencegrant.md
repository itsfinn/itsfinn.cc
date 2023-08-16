---
title: "ReferenceGrant - Kubernetes Gateway API"
date: 2023-08-16T12:00:00+08:00
isCJKLanguage: true
Description: "ReferenceGrant - Kubernetes Gateway API"
Tags: ["k8s", "kubernetes", "ReferenceGrant"]
Categories: ["k8s", "kubernetes"]
DisableComments: false
---


# ReferenceGrant

> Note
> ---
> 该资源最初命名为 `ReferencePolicy`。它被重命名为 `ReferenceGrant` 以避免与 `PolicyAttachment` 混淆。

`ReferenceGrant` 可用于在 Gateway API 中启用跨命名空间引用。特别是，路由可以将流量转发到其他命名空间中的后端，或者网关可以引用另一个命名空间中的 `Secret`。

![Reference Grant](https://gateway-api.sigs.k8s.io/images/referencegrant-simple.svg)

过去，我们已经看到跨命名空间边界转发流量是一项理想的功能，但如果没有像 `ReferenceGrant` 这样的保护措施，就会出现漏洞。

如果从其名称空间外部引用某个对象，则该对象的所有者必须创建 `ReferenceGrant` 资源以显式允许该引用。如果没有 `ReferenceGrant`，跨命名空间引用是无效的。

## Structure
基本上，`ReferenceGrant` 由两个列表组成，一个是可能来自的资源引用列表，另一个是可以被引用的资源列表。

`from` 列表允许你指定资源的组、种类和命名空间，这些资源可以引用 `to` 列表中描述的项。

`to` 列表允许您指定 `from` 列表中描述的项目可能引用的资源组和类型。 `to` 列表中不需要命名空间，因为 `ReferenceGrant` 只能用于允许引用与 `ReferenceGrant` 位于同一命名空间中的资源。

## Example

以下示例显示命名空间 `foo` 中的 `HTTPRoute` 如何引用命名空间 `bar` 中的服务。在此示例中， `bar` 命名空间中的 `ReferenceGrant` 显式允许从 `foo` 命名空间中的 `HTTPRoutes` 引用服务。

```yaml
kind: HTTPRoute
metadata:
  name: foo
  namespace: foo
spec:
  rules:
  - matches:
    - path: /bar
    backendRefs:
      - name: bar
        namespace: bar
---
kind: ReferenceGrant
metadata:
  name: bar
  namespace: bar
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: foo
  to:
  - group: ""
    kind: Service
```

## API 设计决策

虽然 API 本质上很简单，但它有一些值得注意的决定：

1. 每个 `ReferenceGrant` 仅支持单个 `From` 和 `To` 部分。额外的信任关系必须使用额外的 `ReferenceGrant` 资源来建模。

2. 资源名称有意从 `ReferenceGrant` 的 `From` 部分中排除，因为它们很少提供任何有意义的保护。能够写入命名空间内某种资源的用户, 始终可以重命名资源, 或更改资源的结构以匹配给定的授权。

3. 每个 `From` 结构允许有一个命名空间。尽管使用 `selector` 会更强大，但它会鼓励不必要的不​​安全配置。

4. 这些资源的效果纯粹是相加的，它们相互叠加。这使得他们之间不可能发生冲突。


## 实施指南

此 API 依赖于运行时验证。实现必须监视这些资源的更改，并在每次更改或删除后重新计算跨命名空间引用的有效性。

当传达跨命名空间引用的状态时，实现**不得**公开有关另一个命名空间中资源存在的信息，除非存在允许引用发生的 `ReferenceGrant`。这意味着，如果在没有 `ReferenceGrant` 的情况下对不存在的资源进行跨命名空间引用，则任何状态条件或警告消息都需要关注以下事实：`ReferenceGrant` 不存在以允许此引用。不应提供有关引用资源是否存在的提示。

## 例外情况

跨命名空间路由 -> 网关绑定遵循略有不同的模式，其中握手机制内置于网关资源中。有关该方法的更多信息，请参阅相关的安全模型文档。尽管在概念上与 `ReferenceGrant` 类似，但此配置直接内置到网关侦听器中，并允许对每个侦听器进行细粒度配置，而这是 `ReferenceGrant` 无法实现的。

在某些情况下，忽略 `ReferenceGrant` 而采用其他安全机制可能是可以接受的。只有当 `NetworkPolicy` 等其他机制可以通过实现有效限制跨命名空间引用时，才可以这样做。

选择做出此例外的实现必须清楚地记录其实现不遵守 `ReferenceGrant`,详细说明可用的替代保护措施。注意，这个功能不太可能适用于`API`的`ingress`实现，也不太可能适用于所有mesh实现。

有关跨命名空间引用涉及的风险的示例，请参阅 [CVE-2021-25740](https://github.com/kubernetes/kubernetes/issues/103675)。此 `API` 的实现需要非常小心，以避免混淆代理攻击。 `ReferenceGrant` 为此提供了保障。只有在绝对确定其他同等有效的保障措施到位的情况下才可以例外。

## 一致性级别

对于源自以下对象的跨命名空间引用, 支持 `ReferenceGrant` 是一个 "核心" 的一致性要求:

- Gateway
- GRPCRoute
- HTTPRoute
- TLSRoute
- TCPRoute
- UDPRoute

.
也就是说，所有实现都必须将此流程用于网关和任何核心 xRoute 类型中的任何跨命名空间引用，除非上面的 [列外](#例外情况) 部分中指出。

其他 "特定实现" 级别的对象和引用, 也必须使用此流进行跨命名空间引用，除非上面 [列外](#例外情况) 部分中指出。

## 未来 API 组可能发生的变化

`ReferenceGrant` 开始在 ` Gateway API` 和 `SIG Network` 用例之外引起人们的兴趣。这个资源可能会转移到一个更中立的家。将来某个时候，`ReferenceGrant` API 的用户可能需要转换到不同的 API 组（而不是 `gateway.networking.k8s.io` ）。
