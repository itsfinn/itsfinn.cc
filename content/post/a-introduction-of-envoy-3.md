---
title: "Envoy 介绍之(三)"
date: 2023-12-26T12:00:00+08:00
isCJKLanguage: true
Description: "Envoy 官网介绍文档的中文翻译"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---

原文地址: https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/intro

Envoy 官网介绍文档的中文翻译(上游集群部分): 聚合集群、异常检测、熔断机制
<!--more-->


## 聚合集群

聚合集群功能用于在配置不同的多个集群之间实现无缝的故障转移，比如可以在EDS上游集群与STRICT_DNS上游集群之间，
或是在使用ROUND_ROBIN负载均衡策略的集群与使用MAGLEV策略的集群之间，亦或是在连接超时设为0.1秒的集群与设为
1秒的集群之间进行切换。这种聚合是通过在[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/clusters/aggregate/v3/cluster.proto#envoy-v3-api-msg-extensions-clusters-aggregate-v3-clusterconfig)文件中指明集群名称来实现的，同时各集群的回退优先级是根据它们在[集群列表](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/extensions/clusters/aggregate/v3/cluster.proto#envoy-v3-api-field-extensions-clusters-aggregate-v3-clusterconfig-clusters)中
的顺序来隐式确定的。聚合集群采用了层次化的负载均衡策略，负载均衡器会先确定要使用的集群和其优先级，然后再把负载
均衡的任务转交给该集群的负载均衡器。在此过程中，顶级的负载均衡器通过整合多个集群的优先级设置，来复用已有的负载
均衡算法，从而实现高效的资源分配。

### 优先级线性化

在负载均衡的过程中，为了简化服务节点的选择，上游服务节点会被分为多个优先级，每个优先级都包含了健康的、性能下降的
和不健康的节点。优先级线性化通过整合来自不同集群的优先级，以简化这一过程。举个例子，如果主集群设定了三级优先级，
次级集群设定了两级，三级集群也设定了两级，那么在发生故障转移时，节点选择的优先顺序将依次是主集群、次级集群和三级
集群。

| Cluster | Priority Level | Priority Level after Linearization |
| :--: | :--: | :--: |
| Primary | 0 | 0 |
| Primary | 1 | 1 |
| Primary | 2 | 2 |
| Secondary | 0 | 3 |
| Secondary | 1 | 4 |
| Tertiary | 0 | 5 |
| Tertiary | 1 | 6 |

### 示例

以下是一个聚合集群的配置示例：

```yaml
name: aggregate_cluster
connect_timeout: 0.25s
lb_policy: CLUSTER_PROVIDED
cluster_type:
  name: envoy.clusters.aggregate
  typed_config:
    "@type": type.googleapis.com/envoy.extensions.clusters.aggregate.v3.ClusterConfig
    clusters:
    # cluster primary, secondary and tertiary should be defined outside.
    - primary
    - secondary
    - tertiary
```

注意：由于聚合负载均衡器会在负载均衡过程中重写优先级负载（PriorityLoad），因此 [PriorityLoad 重试插件](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-retrypolicy-retry-priority) 在聚合集群
中将不起作用。

### 负载均衡示例

聚合集群采用了分层负载均衡算法，其中最高层会根据每个集群内各优先级的整体健康状况来分配流量。在本示例中，聚合
集群由两个不同的集群组成，这与之前提到的配置所描述的情况有所区别。

![envoy loadbalancing example](envoy-loadbalancing-exmaple.png)

注意事项：上述负载均衡策略采用了默认的[超额配给因子](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/load_balancing/overprovisioning#arch-overview-load-balancing-overprovisioning-factor)1.4。这表示，即便某个优先级中只有80%的节点处于健康状态，
在80 * 1.4 > 100的计算下，这个优先级也能被认定为完全健康。

该示例阐明了聚合集群级别的负载均衡器是如何根据健康度选择集群的。比如，健康度为{{20, 20, 10}, {25, 25}}
的情况下，计算出的优先级负载分配为{{28%, 28%, 14%}, {30%, 0%}}。当总体健康度降至100以下时，
流量会按照规范化后的健康度分配，使得各个优先级的健康度总和达到100以下。例如，健康度为{{20, 0, 0}, {20, 0}}
（规范化后的总体健康度为56）的情况会被规范化，每个集群将分别接收20 * 1.4 / 56 = 50%的流量，
由此计算得出的优先级负载分配为{{50%, 0%, 0%}, {50%, 0%, 0%}}。

负载均衡器利用优先级逻辑来辅助集群的选择。这种逻辑以整数形式表示健康度分数，一个优先级的健康度分数是
该优先级中健康节点的百分比乘以超额配给因子，且该值上限为100%。在流量分配中，P=0级别的节点将获得其健康
度分数所占的流量百分比，其余流量则流向P=1级别的节点（这里假设P=1级别是完全健康的，后续还会详细说明）。
各集群获得的流量百分比总和称为系统的“集群优先级负载”。例如，在主集群中，如果P=0级别有20%的节点健康，
P=1级别有20%的节点健康，P=2级别有10%的节点健康；在次级集群中，P=0级别有25%的节点健康，P=1级别也有25%
的节点健康。那么，主集群将接收70%的流量，计算方式为20% * 1.4 + 20% * 1.4 + 10% * 1.4。次级集群将接收
30%的流量，计算方式为min(100 - 70, 25% * 1.4 + 25% * 1.4)。所有集群接收的流量加起来总共是100%。在进行
集群和优先级选择之前，会预先计算出规范化的健康度分数和优先级负载。

简化的算法伪代码如下：

```shell
health(P_X) = min(100, 1.4 * 100 * healthy_P_X_backends / total_P_X_backends), where
                total_P_X_backends is the number of backends for priority P_X after linearization
normalized_total_health = min(100, Σ(health(P_0)...health(P_X)))
cluster_priority_load(C_0) = min(100, Σ(health(P_0)...health(P_k)) * 100 / normalized_total_health),
                where P_0...P_k belong to C_0
cluster_priority_load(C_X) = min(100 - Σ(priority_load(C_0)..priority_load(C_X-1)),
                         Σ(health(P_x)...health(P_X)) * 100 / normalized_total_health),
                         where P_x...P_X belong to C_X
map from priorities to clusters:
  P_0 ... P_k ... ...P_x ... P_X
  ^       ^          ^       ^
  cluster C_0        cluster C_X
```

在负载均衡的第二阶段，系统会将任务委派给第一阶段选择的集群，而这个集群可以采用负载均衡器类型所指定的任意算法来执行负载均衡。

## 异常检测

异常检测及剔除是指，动态识别上游集群中某些与众不同的节点，并将这些节点从健康的[负载均衡](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/load_balancing/overview#arch-overview-load-balancing)池中移除的过程。
这种差异可能体现在多个方面，如连续失败次数、一定时间内的成功率、响应延迟等。异常检测属于被动健康检查方式。
Envoy 同时也提供了[主动健康检查](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/health_checking#arch-overview-health-checking)功能。被动和主动健康检查可以一起启用，也可以分别独立使用，它们共同构成了全面的
上游健康检查方案。异常检测是[集群配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-msg-config-cluster-v3-outlierdetection)的一部分，并且它依赖于过滤器来报告错误、超时和连接重置情况。目前，支持异常
检测的过滤器包括：[HTTP router](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/router_filter#config-http-filters-router),
[TCP proxy](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/tcp_proxy_filter#config-network-filters-tcp-proxy), 
[Redis proxy](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/redis_proxy_filter#config-network-filters-redis-proxy) 
和 [Thrift proxy](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/thrift_proxy_filter#config-network-filters-thrift-proxy)。

检测到的错误可以分为外部产生的和本地产生的两类。外部产生的错误与特定的请求有关，这些错误发生在上游服务器，
是对接收到的请求做出的响应。例如，HTTP服务器返回500错误码，或Redis服务器返回一个无法解码的数据包。这些
错误在Envoy成功建立连接后由上游主机生成。本地产生的错误是由Envoy在遇到中断或妨碍与上游主机通信的情况时
产生的。本地产生的错误包括超时、TCP连接重置、无法连接到特定端口等情况。

检测到的错误类型依赖于过滤器的类型。例如，[HTTP router](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/router_filter#config-http-filters-router) filter 能检测到本地产生的错误（如超时、连接重置等与
上游主机连接相关的问题），同时由于其能够解析HTTP协议，也能报告HTTP服务器返回的外部产生的错误。在这种情形下，
即便与上游HTTP服务器的连接建立成功，与服务器的通信交互仍然可能会失败。相反，[TCP proxy](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/listeners/network_filters/tcp_proxy_filter#config-network-filters-tcp-proxy) filter 不处理TCP协议
以上的内容，因此它只报告本地产生的错误。

在默认配置（[outlier_detection.split_external_local_origin_errors](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-split-external-local-origin-errors) 设为 false）下，本地产生的错误和外部产生的
错误不会被区分开，它们都被归入同一个统计类别中，并且会根据 [outlier_detection.consecutive_5xx](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-consecutive-5xx)、
[outlier_detection.consecutive_gateway_failure](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-consecutive-gateway-failure) 
和 [outlier_detection.success_rate_stdev_factor](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-success-rate-stdev-factor)
配置项进行评估。比如，如果一个上游HTTP服务器因为超时而导致连接失败两次，随后在连接成功后服务器返回了500错误码，那么总的错误计数
就会是3次。

> **注意**
>
> 对TCP流量来说，设置 outlier_detection.consecutive_5xx 参数是正确的选择，此参数直接关联到TCP连接失败的情况。

异常检测的配置还可以区分本地产生的错误和外部产生的错误。这通过 [outlier_detection.split_external_local_origin_errors](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-split-external-local-origin-errors)
配置项来实现。在这种模式下，本地产生的错误和外部产生的错误将由不同的计数器进行跟踪，异常检测器可以根据需要配置为仅
响应本地产生的错误或仅响应外部产生的错误。

需要明白的是，一个集群可能被多个过滤器链共同使用。如果某个过滤器链根据其配置的异常检测逻辑剔除了一个主机，那么即便其他过滤器链
的异常检测配置不会导致剔除该主机，这个主机的剔除行为也会影响到其他过滤器链。

### 剔除算法

剔除操作的执行方式取决于所使用的异常检测类型，它可能是即时执行的（如在连续返回5xx错误时），也可能是周期性执行的（如在
定期检查成功率时）。剔除算法按以下流程进行：

1. 系统识别一个节点为异常节点。

2. 系统会检查当前被剔除的节点数量是否低于设定的阈值（由 [outlier_detection.max_ejection_percent](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-max-ejection-percent)
配置项设定）。如果已被剔除的节点数量过多，则不会剔除当前节点。

3. 被确定为异常的节点将被暂时剔除一段时间。所谓剔除，是指节点被标记为不健康状态，在负载均衡过程中不会被选用，
除非处于紧急情况。剔除的时间长度是 [outlier_detection.base_ejection_time](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-base-ejection-time)
配置值乘以节点连续被剔除的次数。这意味着如果节点持续出现问题，它被剔除的时间将越来越长。一旦达到
[outlier_detection.max_ejection_time](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-max-ejection-time)
设定的最大值后，剔除时间不再增加。当节点恢复健康状态时，其被剔除的时间会逐渐减少。节点的健康状态会按 [outlier_detection.interval](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-interval)
设定的时间间隔进行检查，如果检查结果是健康的，则剔除的时间会缩短。假设节点保持健康，剔除时间减少到最小值 outlier_detection.base_ejection_time
所需的大致时间是 outlier_detection.max_ejection_time / outlier_detection.base_ejection_time * outlier_detection.interval 秒。

4. 一旦节点满足剔除时间，它将自动重新纳入服务。通常，异常检测会与主动健康检查配合使用，共同构成一个全面的健康检查方案。

### 检测类型

Envoy 支持以下几种异常检测类型：

#### 连续5xx

在默认模式（[outlier_detection.split_external_local_origin_errors](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-split-external-local-origin-errors) 
设为 false）中，连续5xx检测类型会同时考虑所有类型的错误，包括本地产生和外部产生的交易错误。

> **注意**
>
> 由非HTTP过滤器产生的错误，如 TCP proxy 或 Redis proxy，会在内部被视为HTTP 5xx错误并相应处理。

在错误分离模式（[outlier_detection.split_external_local_origin_errors](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-split-external-local-origin-errors) 
设为 true）中，连续5xx检测类型仅考虑外部产生的错误，而忽略本地产生的错误。若上游主机是HTTP服务器，则只有5xx类型的错误会被计入
（关于例外情况，参见 [Consecutive Gateway Failure](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/outlier#consecutive-gateway-failure)）。
对于通过 Redis proxy 服务的Redis服务器，只有来自服务器的格式不正确的响应会被计入。
格式正确的响应，即便包含操作错误（如“索引未找到”或“访问被拒”），也不会被考虑。

如果上游主机返回一定数量的连续5xx类型错误，则会触发剔除机制。
触发剔除所需的连续5xx错误数量由 [outlier_detection.consecutive_5xx](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-consecutive-5xx) 参数控制。

#### 连续网关失败

在默认模式（[outlier_detection.split_external_local_origin_errors](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-split-external-local-origin-errors) 
设为 false）中，连续网关失败检测类型会考虑一部分特定的5xx错误，
这部分错误被称作网关错误（具体是502、503或504状态码），同时也会考虑由本地产生的失败情况，比如连接超时或TCP连接重置等。

在错误分离模式（outlier_detection.split_external_local_origin_errors 设为 true）中，连续网关失败检测类型仅考虑被称作网关错误的5xx错误子集
（502、503或504状态码），这种检测仅由 [http router](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/http/http_filters/router_filter#config-http-filters-router) 过滤器支持。

如果上游主机返回一系列连续的网关错误（502、503或504状态码），将触发剔除机制。触发剔除所需的连续网关失败次数由 [outlier_detection.consecutive_gateway_failure](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-consecutive-gateway-failure) 参数决定。

#### 连续本地起源故障

当 [outlier_detection.split_external_local_origin_errors](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-split-external-local-origin-errors) 
设为 true 时，才会启用连续本地起源故障检测类型。该检测类型专门针对本地产生的故障
（如连接超时、TCP连接重置等）。如果 Envoy 多次无法建立与上游主机的连接，或者与上游主机的通信不断出现中断，则会将上游主机剔除。此检测能够识别
多种本地产生的故障，包括连接超时、TCP连接重置、ICMP通信错误等。触发剔除所需的连续本地起源故障次数由 [outlier_detection.consecutive_local_origin_failure](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-consecutive-local-origin-failure)
参数控制。HTTP router, TCP proxy, 和 Redis proxy 过滤器均支持此种故障检测。

#### 成功率

基于成功率的异常检测收集并汇总集群中每个节点的成功率数据。随后，在设定的时间间隔内，根据统计学的异常检测方法来决定是否剔除节点。
如果节点在统计周期内的请求量小于 [outlier_detection.success_rate_request_volume](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-success-rate-request-volume)
设定值，则不会对该节点进行成功率异常检测。
另外，如果一个集群中符合最低请求量要求的节点数少于 [outlier_detection.success_rate_minimum_hosts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-success-rate-minimum-hosts)
设定值，则不会对该集群执行成功率异常检测。

在默认配置模式（outlier_detection.split_external_local_origin_errors 设为 false）中，成功率检测会考虑所有类型的错误，包括本地产生的和外部产生的错误。
此模式下，[outlier_detection.enforcing_local_origin_success](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-enforcing-local-origin-success-rate) 配置项不生效。

在错误分离模式（[outlier_detection.split_external_local_origin_errors](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-split-external-local-origin-errors) 
设为 true）中，本地产生的错误和外部产生的错误会被分别计数和处理。
大部分相关配置项，如 
[outlier_detection.success_rate_minimum_hosts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-success-rate-minimum-hosts)
、 [outlier_detection.success_rate_request_volume](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-success-rate-request-volume)
和 [outlier_detection.success_rate_stdev_factor](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-success-rate-stdev-factor)，
适用于这两种错误类型，但 
[outlier_detection.enforcing_success_rate](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-enforcing-success-rate)
仅适用于外部产生的错误，而 
[outlier_detection.enforcing_local_origin_success_rate](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-enforcing-local-origin-success-rate)
仅适用于本地产生的错误。

#### 失败百分比

基于失败百分比的异常检测与成功率检测的工作原理类似，都依赖于从集群中各个节点收集的成功率统计数据。不同之处在于，它不是将节点的成功率与
集群整体的平均成功率进行比较，而是与用户设定的固定阈值比较。这个固定阈值通过 
[outlier_detection.failure_percentage_threshold](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-failure-percentage-threshold)
参数来设定。

其他用于失败百分比检测的配置项与成功率检测的配置项相似。失败百分比检测同样遵从 
[outlier_detection.split_external_local_origin_errors](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-split-external-local-origin-errors) 
设置；
针对外部和本地产生的错误的应用百分比分别由 
[outlier_detection.enforcing_failure_percentage](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-enforcing-failure-percentage)
和
[outlier_detection.enforcing_failure_percentage_local_origin](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-enforcing-failure-percentage-local-origin)
参数控制。和成功率检测相同，如果节点在统计周期内的请求量低于
[outlier_detection.failure_percentage_request_volume](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-failure-percentage-request-volume)
设定值，则不会对该节点进行失败百分比异常检测。同样，如果一个集群中满足最低请求量要求的
节点数少于 
[outlier_detection.failure_percentage_minimum_hosts](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-failure-percentage-minimum-hosts)
设定值，那么也不会对该集群执行失败百分比异常检测。

### gRPC

在处理gRPC请求时，系统将根据grpc-status响应头转换成的HTTP状态来执行异常点检测。

> **注意**
>
> 如果同时配置了[主动健康检查](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/health_checking#arch-overview-health-checking)，
> 一旦健康检查成功，系统会将主机重新加入集群，并重置所有异常点检测的计数器。
> 如果您的健康检查并未针对数据平面流量进行验证，则可能出现健康检查通过而实际流量处理失败的情况，
> 导致端点被过早地重新加入集群。若需关闭这一功能，请将
> [outlier_detection.successful_active_health_check_uneject_host](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-field-config-cluster-v3-outlierdetection-successful-active-health-check-uneject-host)
> 配置选项设为`false`。

### 驱逐事件日志

Envoy能够提供可选的驱逐事件日志，这在日常运维中非常实用，因为仅有的全局统计信息并不足以说明哪些主机因何原因被驱逐。
这些日志以protobuf格式记录[OutlierDetectionEvent](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/data/cluster/v3/outlier_detection_event.proto#envoy-v3-api-msg-data-cluster-v3-outlierdetectionevent)
事件信息，您可以在集群管理器的[异常检测配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-field-config-bootstrap-v3-clustermanager-outlier-detection)中启用驱逐事件日志。

### 配置参考

- 集群管理器的[全局配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/bootstrap/v3/bootstrap.proto#envoy-v3-api-field-config-bootstrap-v3-clustermanager-outlier-detection)
- 面向单个集群的[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/outlier_detection.proto#envoy-v3-api-msg-config-cluster-v3-outlierdetection)
- 运行时[配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cluster_runtime#config-cluster-manager-cluster-runtime-outlier-detection)
- 统计信息[参考](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cluster_stats#config-cluster-manager-cluster-stats-outlier-detection)

## 断路器

在分布式系统中，断路器扮演着至关重要的角色。它几乎总是能够使系统快速地失败并且尽快对下游系统施加背压。
Envoy 服务网格的一个主要优势在于，它能在网络层面强制实施电路断路限制，这样就无需为每个应用单独配置和编写代码了。
Envoy 支持多种类型的完全分布式（不协调）断路功能：

- 集群最大连接数：Envoy 为上游集群中的所有主机建立的连接数量上限。如果达到了这个上限，集群的 upstream_cx_overflow
  计数器就会增加。不论是活动状态还是正在关闭的连接，都会被包括在这个限制之内。即便达到了这个上限，Envoy 仍会确保
  经过集群负载均衡选定的主机至少能分配到一个连接。这就意味着，集群的 upstream_cx_active 计数可能会超过集群最大连接数
  的断路器限制，最大不会超过`集群最大连接数` + (集群中端点的数量) * (集群的连接池数量)。这个总数是针对所有工作线程中的
  连接而言的。关于一个集群可能有多少个连接池，请参考[连接池](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/upstream/connection_pooling#arch-overview-conn-pool-how-many)部分。

- 集群最大挂起请求：等待一个可用的连接池连接时，可以排队的最大请求数。当上游没有足够的连接可供立即处理请求时，
  这些请求会被加入到挂起请求的队列。对于 HTTP/2 连接，如果未设置
  [最大并发流](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-http2protocoloptions-max-concurrent-streams)和
  [每个连接的最大请求数](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-field-config-cluster-v3-cluster-max-requests-per-connection)，
  所有请求将通过同一连接多路复用，这种情况下，只有在没有连接建立时才会触发此电路断路器。当该电路断路器溢出时，集群的
  [upstream_rq_pending_overflow](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cluster_stats#config-cluster-manager-cluster-stats)
  计数器也会相应增加。对于 HTTP/3，与 HTTP/2 的
  [最大并发流](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-quicprotocoloptions-max-concurrent-streams)
  相对应的参数也是
  [最大并发流](https://www.envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/core/v3/protocol.proto#envoy-v3-api-field-config-core-v3-quicprotocoloptions-max-concurrent-streams)。

- 集群最大请求：在任何时刻，集群中所有主机的最大未处理请求数。当这个限制被超出时，同样会导致集群的
  [upstream_rq_pending_overflow](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cluster_stats#config-cluster-manager-cluster-stats) 计数器增加。

- 集群最大挂起重试数：在任何时刻，集群中所有主机允许的最大挂起重试操作数量。我们通常建议使用重试预算来控制重试行为；
  但如果更偏向于采用静态的电路断路策略，那么应当更为严格地控制重试次数。这是为了确保在允许因偶发故障而进行的重试的同时，
  避免重试次数急剧增加并引发大规模的连锁故障。当该电路断路器达到极限时，集群的 upstream_rq_retry_overflow 计数器会递增。

- 集群最大并发连接池数：可以同时被创建的连接池的最大数量。某些功能，比如
  [Original Src Listener Filter](https://www.envoyproxy.io/docs/envoy/v1.28.0/intro/arch_overview/other_features/ip_transparency#arch-overview-ip-transparency-original-src-listener)，
  可能会创建大量的连接池。当集群的并发连接池数量达到上限时，它会试图回收一个闲置的连接池。如果无法回收，
  则电路断路器将会触发。这和集群最大连接数的不同之处在于，连接池没有超时机制，而连接通常会有超时机制。
  连接会自动被清理，而连接池则不会。需要注意的是，要使连接池正常工作，至少需要一个上游连接，因此这个值通常不应超过
  集群最大连接数。如果电路断路器触发，集群的 upstream_cx_pool_overflow 计数器将会递增。

每个断路器的限制都是[可配置](https://www.envoyproxy.io/docs/envoy/v1.28.0/configuration/upstream/cluster_manager/cluster_circuit_breakers#config-cluster-manager-cluster-circuit-breakers)
的，并且会根据每个上游集群以及各自的优先级进行跟踪。这样做可以使得分布式系统的不同部分
能够进行独立调优，同时设置有不同的限制值。这些断路器的当前状态，例如距离断路器触发前的可用资源数量，都可以通过统计数据查看。

工作线程之间共享电路断路器的限制，比如，如果活动连接的阈值设定为500，工作线程1已经有498个活动连接，那么工作线程2只能再分配2个连接。
由于实现最终会保持一致，线程间的竞争可能导致这些限制被暂时超出。

断路器默认是开启状态，并设置了一定的默认值，例如每个集群默认1024个连接。
若需要禁用电路断路器，可将各项阈值设置为系统允许的最大值。

需要注意的是，当触发断路时，如果是HTTP请求，路由过滤器会在响应中设置 x-envoy-overloaded 头部信息。
