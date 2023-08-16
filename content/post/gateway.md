---
title: "Gateway - Kubernetes Gateway API"
date: 2023-08-16T12:00:00+08:00
isCJKLanguage: true
Description: "Gateway - Kubernetes Gateway API"
Tags: ["k8s", "kubernetes", "Gateway"]
Categories: ["k8s", "kubernetes"]
DisableComments: false
---


# Gateway

`Gateway` 与基础设施配置的生命周期是1:1的。当用户创建 `Gateway` 时， `GatewayClass` 控制器会配置或配置一些负载平衡基础设施（有关详细信息，请参见下文）。 在 Kubernetes Gateway API 中, `Gateway` 是触发此 API 中操作的资源。此 API 中的其他资源是配置片段，直到创建网关将资源链接在一起。

`Gateway` 通过将 `Listener` 绑定到一组 IP 地址来表示处理服务流量的基础设施实例。

`Gateway` 短名称是 `gtw`, 你可以使用如下命令获取命名空间内的 `Gateway` 资源:

```shell
$ kubectl get gtw
```

## Spec

`Gateway` 规范定义了以下内容：

- `GatewayClassName` - 定义此网关使用的 `GatewayClass` 对象的名称。
- `Listeners` - 定义主机名、端口、协议、终止、TLS 设置以及哪些路由可以附加到侦听器。
- `Addresses` - 定义为此网关请求的网络地址。

如果无法实现网关规范中指定的所需配置，网关将处于错误状态，详细信息由状态条件提供。

### 侦听器 Listeners

与此网关关联的侦听器。侦听器定义绑定到该网关地址的逻辑端点。必须至少指定一个监听器。

网关中的每个侦听器必须具有主机名、端口和协议的唯一组合。

在 HTTP 一致性配置文件中，以下端口和协议组合被视为核心且必须受支持：

- Port: 80, Protocol: HTTP
- Port: 443, Protocol: HTTPS

在 TLS 一致性配置文件中，以下端口和协议组合被视为核心且必须受支持：

- Port: 443, Protocol: TLS

上面未列出的端口和协议组合被视为扩展。

如果实现确定组中的侦听器是"兼容的", 则实现可以按端口对侦听器进行分组，然后将每组侦听器折叠成单个侦听器。

实现还可以将属于不同网关的兼容侦听器组合在一起并折叠。

例如，如果满足以下所有条件，则实现可能会认为侦听器彼此兼容：

1. 组内的每个侦听器指定“HTTP”协议，或者组内的每个侦听器指定“HTTPS”或“TLS”协议。
2. 组内的每个侦听器指定一个在组内唯一的主机名。
3. 作为一种特殊情况，组中的一个监听器可以省略主机名，在这种情况下，当没有其他监听器匹配时，该监听器匹配。

如果实现确实折叠了兼容的侦听器，则来自客户端的请求中, 提供的主机名必须与侦听器匹配，以找到正确的路由集。传入的主机名必须使用每个侦听器的主机名字段按照从最具体到最不具体的顺序进行匹配。也就是说，必须在通配符匹配之前处理精确匹配。

如果 Listeners 配置了多个具有相同端口值但不兼容的侦听器，则实现必须在侦听器状态中引发“冲突”条件。

## 部署模型

根据 `GatewayClass` ，创建 `Gateway` 可以执行以下任一操作：

- 使用云API创建LB实例。
- 生成软件 LB 的新实例（在这个或另一个集群中）。
- 将配置片段添加到已实例化的 LB 中以处理新路由。
- 对SDN进行编程以实现配置。
- 还有一些我们还没想到的...

API 未指定将采取其中哪一项操作。

## 网关状态

`GatewayStatus` 用于显示 `Gateway` 相对于 `spec` 中表示的所需状态的状态。 `GatewayStatus` 由以下内容组成：

- `Addresses` - 列出实际绑定到 `Gateway` 的IP地址。
- `Listeners` - 为 spec 中定义的每个唯一侦听器提供状态。
- `Conditions` - 描述 `Gateway` 当前的状态情况。

`Conditions` 和 `Listeners.conditions` 都遵循 Kubernetes 中其他地方使用的条件模式。这是一个列表，其中包括条件类型、条件的状态以及该条件上次更改的时间。
