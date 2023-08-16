---
title: "GatewayClass - Kubernetes Gateway API"
date: 2023-08-16T12:00:00+08:00
isCJKLanguage: true
Description: "GatewayClass - Kubernetes Gateway API"
Tags: ["k8s", "kubernetes", "GatewayClass"]
Categories: ["k8s", "kubernetes"]
DisableComments: false
---


# GatewayClass

`GatewayClass` 是由基础设施提供商定义的集群范围的资源。描述了一类用户可以创建的 `Gateway` 资源。短名称是 `gc`, 你可以使用如下命令获取集群范围内的 `GatewayClass` 资源:

```shell
$ kubectl get gc
```

> 注意：GatewayClass 与 networking.IngressClass 资源具有相同的功能。

```yaml
kind: GatewayClass
metadata:
  name: cluster-gateway
spec:
  controllerName: "example.net/gateway-controller"
```
GatewayClass 描述了一个网关类，用户可以用来创建网关资源。

建议将 `GatewayClass` 作为 `Gateway` 模板使用。这意味着 `Gateway` 基于被创建时 `GatewayClass` 的状态，对 `GatewayClass` 或相关参数的更改不会传播到已经创建出的 `Gateway` 。本建议旨在限制 `GatewayClass` 或相关参数的变化范围。

如果实现选择将 `GatewayClass` 更改传播到现有网关，则实现必须清楚地记录这一点。

每当一个或多个 `Gateway` 使用一个 `GatewayClass` 时，实现**应该**在相关的 `GatewayClass` 上添加 `gateway-exists-finalizer.gateway.network.k8s.io` finalizer。这确保了与网关关联的 `GatewayClass` 在使用时不会被删除。


我们希望基础设施提供者为用户创建一个或多个 `GatewayClass`。它允许将实现 `Gateway` 的机制(例如控制器)与用户解耦。例如，基础设施提供商可以创建两个名为 `internet` 和 `private` 的 `GatewayClass`，以定义面向 internet 和私有的内部应用程序的网关。

```yaml
kind: GatewayClass
metadata:
  name: internet
  ...
---
kind: GatewayClass
metadata:
  name: private
  ...
```

`GatewayClass` 的用户不需要知道 `internet` 和 `private` 是如何实现的。相反，用户只需要了解创建 `Gateway` 时使用的 `GatewayClass` 的结果属性。

## GatewayClass 参数


`Gateway` API 的提供者可能需要将参数作为类定义的一部分传递到其控制器, 这是用 `GatewayClass.spec.parametersRef` 字段完成的。

`ParametersRef` 是对资源的引用，其中包含与 `GatewayClass` 对应的配置参数。如果控制器不需要任何额外的配置，也可以没有这个字段。

`ParametersRef` 可以引用一个标准的 Kubernetes 资源，例如 `ConfigMap`，或者一个特定于实现的自定义资源。资源可以是集群范围内的, 也可以是命名空间范围内的, 对应 `namespace` 字段是可选的, 当引用集群范围的资源时, 一定不可以设置 `namespace` 字段。

如果 `ParametersRef` 字段引用的资源不存在, `GatewayClass` 的 `InvalidParameters` 状态条件将为 `true`。

```yaml
# GatewayClass for Gateways that define Internet-facing applications.
kind: GatewayClass
metadata:
  name: internet
spec:
  controllerName: "example.net/gateway-controller"
  parametersRef:
    group: example.net/v1alpha1
    kind: Config
    name: internet-gateway-config
    namespace: default
  description: "more details about this GatewayClass"
---
apiVersion: example.net/v1alpha1
kind: Config
metadata:
  name: internet-gateway-config
spec:
  ip-address-pool: internet-vips
  ...
```

鼓励使用定制资源来配置 GatewayClass.spec.parametersRef, 但如果需要, 也可以使用 ConfigMap。


## GatewayClass 状态

`GatewayClass` 必须由提供者进行验证，以确保配置的参数有效。类的有效性将通过 `GatewayClass.status` 通知用户：

```yaml
kind: GatewayClass
...
status:
  conditions:
  - type: Accepted
    status: False
    ...
```

新的  `GatewayClass`  将以设置为 `False` 的 `Accepted` 条件开始。此时控制器还没有看到配置。一旦控制器处理完配置，条件将被设置为 `True` ：

```yaml
kind: GatewayClass
...
status:
  conditions:
  - type: Accepted
    status: True
    ...
```

如果 GatewayClass.spec 中存在错误，则条件将非空, 并包含有关错误的信息。

```yaml
kind: GatewayClass
...
status:
  conditions:
  - type: Accepted
    status: False
    reason: BadFooBar
    message: "foobar" is an FooBar.
```

`conditions` 是此 `GatewayClass` 从控制器获得的当前状态。

控制器应该更倾向于使用值 `Accepted` 作为每个条件的类型来发布条件, 用来表示 `GatewayClass` 是否已被 `spec.controllerName` 字段中请求的控制器接受。

`status` 默认为 `Unknown`, 并且当控制器看到使用其控制器字符串的 `GatewayClass` 时必须设置该字段的值。如果控制器可以支持使用此 `GatewayClass` 的供应网关，则此条件的状态必须设置为`True`。否则，该状态必须设置为`False`。如果状态设置为False，控制器应该设置 `message` 和作为解释的 `reason`。

`reason` 的取值如下:

- "Accepted", 当条件为真时，这个 `reason` 与 `Accepted` 条件一起使用。
- "InvalidParameters", 当 `GatewayClass` 因为 `parametersRef` 字段无效而不被接受时，这个 `reason` 与 `Accepted` 条件一起使用， `message` 中应包含更多细节。
- "Pending", 当被请求的控制器还没有决定是否接纳 `GatewayClass` 时，这个原因与 `Accepted` 条件一起使用。这是新版 `GatewayClass` 的默认原因(以前版本是 "Waiting")。


## GatewayClass 控制器选择

`GatewayClass.spec.controllerName` 字段确定负责管理 `GatewayClass` 的控制器实现。该字段的格式是不透明的并且特定于特定控制器。给定控制器字段选择的 `GatewayClass` 取决于集群中各个控制器如何解释该字段。

建议控制器作者/部署, 通过使用在其管理控制下的, 域/路径的组合, 来使他们的选择唯一（例如，管理以 example.net 开头的所有 controller 的控制器是 example.net 域）以避免冲突。

该字段是不可更改的, 且不能为空。

对于控制器版本控制, 可以通过将控制器的版本编码到路径部分来完成。一个示例方案可以是（类似于容器 URI）：

取值长度1-253个字符, 正则表达式: `^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*\/[A-Za-z0-9\/\-._~%!$&'()*+,;=:]+$`

```
example.net/gateway/v1   // Use version 1
example.net/gateway/v2.1 // Use version 2.1
example.net/gateway      // Use the default version
```

## 参考

[1] [GatewayClass - Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/)

[2] [API specification - Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/references/spec/)
