---
title: "Envoy 介绍之(六)"
date: 2023-12-29T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---

原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(操作与配置):xDS配置Api、初始化、运行时配置、热重启、
<!--more-->

# xDS配置API概览

Envoy的架构设计使得可以采用不同的配置管理方式。部署中所采用的方
式将依据实施者的具体需求而定。可以使用完全静态的配置来实现简单
的部署。更复杂的部署则可能会逐渐引入更为复杂的动态配置，这样做的
缺点是实施者需要提供一个或多个基于gRPC/REST的外部配置提供API。
这些API总称为“xDS”（*发现服务）。本文档提供了目前可用的配置选
项的概览。

- 顶层配置[参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/configuration#config)。
- [参考配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/deployment_types/deployment_types#intro-deployment-types)。
- Envoy [v3 API概览](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/overview/overview#config-overview)。
- [xDS API端点](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/overview/xds_api#config-overview-management-server)。

## 完全静态配置

在完全静态配置中，实施者需要提供一系列的[监听器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/listeners#config-listeners)
（及其[过滤器链](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-msg-config-listener-v3-filter)）、
[集群](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cluster_manager#config-cluster-manager)
等配置。动态主机发现只能通过基于DNS的服务发现来实现。配置的重
新加载必须通过内置的[热重启](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/operations/hot_restart#arch-overview-hot-restart)机制来完成。

尽管方法简单，但使用静态配置和优雅的热重启，仍可以创建相对复杂的
部署环境。

## EDS(端点发现服务)

[Endpoint Discovery Service (EDS) API ](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/service_discovery#arch-overview-service-discovery-types-eds)
为Envoy提供了一种更高级的机制，用以发现上游集群的成员。在
静态配置的基础上，EDS使得Envoy部署能够绕过DNS的一些限制（例如响
应中的记录最大数量等），并能够利用更多的信息进行负载均衡和路由（
如金丝雀状态、区域信息等）。

## CDS(集群发现服务)

[Cluster Discovery Service (CDS) API](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cds#config-cluster-manager-cds)
为Envoy提供了一种机制，使其可以在路由过程中发现用到的上游
集群。Envoy将根据API的规定，优雅地添加、更新和移除集群。这个API
使得实施者可以构建一个拓扑结构，在初始配置时Envoy不必了解所有上游
集群。通常，当使用CDS结合HTTP路由（并非使用路由发现服务）时，实
施者会使用路由器的能力，将请求转发到[HTTP请求头](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-routeaction-cluster-header)
中指定的集群。

尽管可以在不使用EDS的情况下通过指定完全静态的集群来使用CDS，但我
们仍然建议对通过CDS指定的集群使用EDS API。在内部，当集群定义发生
更新时，该操作会平滑地进行。然而，所有现有的连接池都需要被排空并重
新连接。EDS则没有这个限制。当通过EDS添加和移除主机时，集群中现有
的主机不会受到影响。

## RDS

[路由发现服务（RDS）API](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/rds#config-http-conn-man-rds)
为Envoy提供了一种机制，使其能够在运行时为HTTP连接管理器过滤器动态发现完整的路由配置。
新的路由配置会在不中断现有请求的前提下平滑替换。该API结合EDS和CDS使用时，使开发者能够构建复杂的路由架构，
如[流量转移](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/traffic_splitting#config-http-conn-man-route-table-traffic-splitting)
、蓝绿部署等。

## VHDS

[虚拟主机发现服务（VHDS）](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/vhds#config-http-conn-man-vhds)
使得可以按需独立于路由配置本身，请求路由配置中的虚拟主机。这个API通常应用于路由配置中包含
大量虚拟主机的部署场景。

## SRDS

[作用域路由发现服务（SRDS）API](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/http/http_routing#arch-overview-http-routing-route-scope)
允许将一个路由表拆分为多个部分。该API通常应用于路由表庞大到无法使用简单线性搜索的
HTTP路由部署场景。

## LDS

[监听器发现服务（LDS）API](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/lds#config-listeners-lds)
为Envoy提供了一种在运行时动态发现监听器的机制，包括整个过滤器栈的配置，
甚至是包含了对RDS引用的HTTP过滤器。引入LDS后，Envoy的几乎所有配置都可以动态化。只有在极少数情况下，
如管理配置、追踪驱动更改、证书更新或程序升级时，才需要进行热重启。

## SDS

[密钥发现服务（SDS）API](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/security/secret#config-secret-discovery-service)
为Envoy提供了一种机制，用于发现监听器所需的加密密钥（包括证书及私钥、TLS会话票据密钥），
以及配置对端证书验证逻辑（信任的根证书、证书吊销列表等）。

## RTDS

[运行时发现服务（RTDS）API](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/operations/runtime#config-runtime-rtds)
使Envoy能够动态获取[运行时层](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/operations/runtime#config-runtime)配置，这一机制相比于文件系统层的配置方式来说，
可能更有优势或可以进行相应的增强。

## ECDS

[扩展配置发现服务（ECDS）API](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/overview/extension#config-overview-extension-discovery)
使得各类扩展配置（比如HTTP过滤器配置）能够与监听器配置分离地进行提供。
这对于那些更适宜独立于主控制平面构建的系统非常有用，例如Web应用防火墙（WAF）、故障测试等场景。

## 聚合xDS(ADS)

EDS、CDS等每个都是一个独立的服务，它们拥有不同的REST/gRPC服务名，如StreamListeners、StreamSecrets。
对于那些需要强制控制不同类型资源到达Envoy的顺序的用户，可以使用聚合xDS，这是一个单一的gRPC服务，
可以在一个gRPC流中传输所有资源类型。（ADS仅支持gRPC）。[关于ADS的更多详细信息](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/overview/xds_api#config-overview-ads)。

## Delta gRPC xDS

标准xDS采用的是“全局状态”模式：每次更新都必须包含所有资源，如果更新中缺少某个资源，则意味着该资源已经被移除。
Envoy还支持xDS（包括ADS）的“delta”变体，即更新仅包含新增、更改或移除的资源。Delta xDS是一种新的协议，
其请求与响应的API与全局状态（State of the World，SotW）模式的API不同。[关于Delta xDS的更多详细信息](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/overview/xds_api#config-overview-delta)。

## xDS TTL

某些xDS更新可能需要设置生存时间（TTL），以此作为控制平面不可用情况下的一种保护措施，相关详细信息请参阅官方文档。
初始化

# 启动初始化

Envoy在启动时的初始化过程相当复杂。本节将概述这一过程的工作原理。在监听器开始监听和接受新连接之前，以下所有步骤都会
先行完成。

- 启动期间，集群管理器将经历多阶段初始化：首先是静态/DNS集群，接着是预定义的EDS集群。如果需要，接下来会初始化CDS，
  并在有限时间内等待响应（或处理失败），然后对CDS提供的集群进行主/备初始化。
- 若集群配置了主动健康检查，Envoy还将执行一轮独立的主动健康检查。
- 集群管理器初始化完成后，将进行RDS和LDS的初始化（如果设置了的话）。服务器将在有限时间内等待至少一个LDS/RDS响应
  （或识别出失败情况）。完成这些步骤后，服务器将开始接收连接。
- 如果LDS本身返回一个监听器，并且该监听器需要一个RDS响应，Envoy会等待一段时间直到收到RDS响应或失败。请注意，每次通过LDS添加新的监听器时，都会发生这个过程，并被称为监听器的预热过程。  
- 在完成所有步骤之后，监听器开始接受新的连接。此流程确保在热重启期间，新的进程在新连接被处理之前完全启动，而旧进程开始关闭。  

初始化设计的一个关键原则是，Envoy始终会在initial_fetch_timeout内完成初始化，并尽量在管理服务器可用性的限制下获取完整的xDS配置。

# 排空

在几种不同的情况下，Envoy会尝试优雅地释放连接。例如，在服务器关闭期间，可以阻止现有请求并设置监听器停止接受新连接，以减少服务器关闭时打开的连接数量。排空行为由服务器选项以及各个监听器配置共同定义。

优雅关闭会在以下几种情况发生：

- 服务器进行[热重启](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/operations/hot_restart#arch-overview-hot-restart)时。
- 服务器通过 [drain_listeners?graceful](https://www.envoyproxy.io/docs/envoy/v1.28.0/operations/admin#operations-admin-interface-drain) 的管理端点开始优雅地关闭序列时。
- 服务器已经通过 [healthcheck/fail](https://www.envoyproxy.io/docs/envoy/v1.28.0/operations/admin#operations-admin-interface-healthcheck-fail) 的管理端点进行手动健康检查并失败时。有关更多信息，请参阅[健康检查过滤器](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/health_checking#arch-overview-health-checking-filter)体系结构概述。
- 监听器正在通过[LDS](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/operations/dynamic_configuration#arch-overview-dynamic-config-lds)进行修改或移除时。

默认情况下，Envoy服务器会在服务器关闭时立即关闭监听器。要在服务器关闭之前的一段时间内排空监听器，请在关闭服务器之前使用[drain_listeners](https://www.envoyproxy.io/docs/envoy/v1.28.0/operations/admin#operations-admin-interface-drain)。监听器将直接停止，没有任何优雅的排空行为，并且立即停止接受新连接。

要在关闭监听器之前添加一个优雅的排空期，请使用查询参数[drain_listeners?graceful](https://www.envoyproxy.io/docs/envoy/v1.28.0/operations/admin#operations-admin-interface-drain)。默认情况下，Envoy会在一段时间内阻止请求（由`--drain-time-s`确定），但会继续接受新连接，直到排空超时。请求阻止的行为由排空管理器确定。

请注意，虽然排空是每个监听器的概念，但它必须在网络过滤器级别上得到支持。目前，唯一支持优雅排空的过滤器是Redis、Mongo、Thrift（如果启用了`envoy.reloadable_features.thrift_connection_draining`运行时功能）和[HTTP连接管理器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/http_conn_man#config-http-conn-man)。

默认情况下，[HTTP连接管理器](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_conn_man/http_conn_man#config-http-conn-man)过滤器会将 "Connection: close" 添加到HTTP1请求中，发送HTTP2 GOAWAY，并在请求完成时终止连接（在延迟关闭期间之后）。

每个[配置的监听器](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/listeners/listeners#arch-overview-listeners)都有一个drain_type设置，用于控制何时进行排空。当前支持的值有：

default

Envoy将在上述三种情况（管理健康检查失败、热重启和LDS更新/删除）下对监听器进行排空。这是默认设置。

modify_only

Envoy将仅在上述第二种和第三种情况（热重启和LDS更新/删除）下对监听器进行排空。如果Envoy托管了入口和出口监听器，则此设置很有用。在尝试进行受控关闭时，将modify_only设置为出口监听器可能很有用，以便它们仅在修改期间进行排空，同时依靠入口监听器的排空来进行完整的服务器排空。
