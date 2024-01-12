---
title: "Envoy 介绍之(九)"
date: 2024-01-14T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---

原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(部署类型)
<!--more-->

Envoy 可以在各种不同场景中使用，但在将其部署为整个基础设施中所有主机的网格时，其最有用。
本节按复杂度递增的顺序描述了三种推荐的部署类型。

# 仅限服务间通信

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/service_to_service.svg)

上图显示了最简单的Envoy部署，该部署使用Envoy作为面向服务的体系结构（SOA）内部所有流量的通信总线。
在此场景中，Envoy暴露出几个侦听器，用于本地原始流量以及服务间通信流量。

## 服务到服务的出站侦听器

这是应用程序用于与基础设施中的其他服务进行通信的端口。
例如，http://localhost:9001。
HTTP和gRPC请求使用HTTP/1.1主机头或HTTP/2或HTTP/3的:authority头来指示请求的目标远程集群。
Envoy根据配置中的详细信息处理服务发现、负载均衡、限流等。
服务只需要了解本地Envoy，无需关注网络拓扑、是否在开发或生产环境中运行等。

此侦听器支持HTTP/1.1或HTTP/2，具体取决于应用程序的功能。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/service_to_service_egress_listener.svg)

## 服务到服务的入口侦听器

这是远程Envoy想要与本地Envoy通信时使用的端口。
例如，http://servicename:9211。
Envoy将传入的请求路由到配置端口上的本地服务。
根据应用程序或负载均衡需求，可能需要多个应用程序端口（例如，如果服务需要HTTP端口和gRPC端口）。
本地Envoy执行所需的缓冲、断路器等操作。

我们的默认配置使用HTTP/2进行所有Envoy到Envoy的通信，
无论应用程序在使用本地Envoy时使用HTTP/1.1还是HTTP/2。
通过长连接和明确的重置通知，HTTP/2比HTTP/1.1提供更好的性能。

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/service_to_service_ingress_listener.svg)

## 可选的外部服务出站侦听器

通常，本地服务想要与外部服务通信时，会为每个外部服务使用一个明确的出站端口。
这是因为一些外部服务的SDK不容易支持覆盖主机头以允许标准的HTTP反向代理行为。
例如，http://localhost:9250可能被分配给与DynamoDB的连接。
我们建议对于所有外部服务使用本地端口路由，而不是对某些外部服务使用主机路由，对其他一些服务使用本地端口路由。

## 集成发现服务

推荐的服务到服务配置使用外部发现服务进行所有集群查找。这为Envoy提供了进行负载平衡、收集统计信息等时所需的最详细信息。

## 配置模板

源代码分发中包含[一个服务到服务配置的示例](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/deployment_types/deployment_types#intro-deployment-types)。

# 服务到服务加前端代理

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/front_proxy.svg)

上图显示了作为HTTP L7边缘反向代理的Envoy集群后面的服务到服务配置。反向代理提供以下功能：

- 终止TLS。
- 支持HTTP/1.1、HTTP/2和HTTP/3。
- 完整的HTTP L7路由支持。
- 通过标准[入口端口](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/deployment_types/service_to_service#deployment-type-service-to-service-ingress)
  与服务到服务的Envoy集群通信，并使用发现服务进行主机查找。
  因此，前端Envoy主机的工作方式与其他任何Envoy主机相同，除了它们不与其他服务共存的事实。
  这意味着它们以相同的方式操作并发出相同的统计数据。

## 配置模板

源代码分发中包含一个前端代理配置示例。请参阅[此处](https://www.envoyproxy.io/docs/envoy/v1.28.0/start/sandboxes/front_proxy#install-sandboxes-front-proxy)了解更多信息。

# 服务到服务、前端代理和双代理

![](https://www.envoyproxy.io/docs/envoy/v1.28.0/_images/double_proxy.svg)

上图显示了与另一个作为双代理运行的Envoy集群一起的
[前端代理](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/deployment_types/front_proxy#deployment-type-front-proxy)
配置。双代理的背后想法是，在离用户尽可能近的地方终止TLS和客户端连接会更有效（更短的TLS握手往返时间，更快的TCP CWND扩展，更少的丢包机会等）。
在双代理中终止的连接然后被复用到在主数据中心运行的HTTP/2或HTTP/3的长期连接上。

在上述图表中，区域1中的前端Envoy代理通过TLS双向认证和固定证书与区域2中的前端Envoy代理进行身份验证。
这允许在区域2中运行的前端Envoy实例信任通常不可信的传入请求的元素（例如x-forwarded-for HTTP头）。

## 配置模板

源代码分发中包含一个双代理配置示例。请参阅[此处](https://www.envoyproxy.io/docs/envoy/v1.28.0/start/sandboxes/double-proxy#install-sandboxes-double-proxy)了解更多信息。
