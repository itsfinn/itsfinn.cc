---
title: "使用 k8s Service 作为集群审计 WebHook 后端收集审计日志"
date: 2023-06-13T12:00:00+08:00
isCJKLanguage: true
Description: "为 k8s 或 k3s 配置使用 k8s Service 作为集群审计 WebHook 后端收集审计日志"
Tags: ["k8s", "kubernetes", "k3s", "Auditing", "审计"]
Categories: ["k8s", "kubernetes", "k3s"]
DisableComments: false
---


# 使用 k8s Service 作为集群审计 WebHook 后端收集审计日志

# 什么是审计?

Kubernetes审计（Auditing）功能提供了与安全相关的、按时间顺序排列的记录集, 记录每个用户、使用 Kubernetes API 的应用以及控制面自身引发的活动。所以这包括了用户发起的调用以及k8s控制面其他模块发起的调用.

审计记录最初产生于kube-apiserver内部。每个请求会被记录多个阶段（stage）。已定义的阶段有：

- RequestReceived- 此阶段对应审计处理器接收到请求后，并且在委托给 其余处理器之前生成的事件。
- ResponseStarted- 在响应消息的头部发送后，响应消息体发送前生成的事件。 只有长时间运行的请求（例如 watch）才会生成这个阶段。
- ResponseComplete- 当响应消息体完成并且没有更多数据需要传输的时候。
- Panic- 当 panic 发生时生成。

审计日志的结果也是一个 Event 或 EventList 结构, 但这跟 k8s 的 Event API 对象不一样, 使用 `kubectl describe pods` 命令看到的的 Events 是一种 API 资源对象, 跟 pod service 一样可以被 kubectl 操作, 审计日志的 event 只是一条日志记录, 字段结构比 Event API 对象 丰富很多

## 创建策略

k8s 已定义的四个审计级别有：

- None- 符合这条规则的日志将不会记录。
- Metadata- 记录请求的元数据（请求的用户、时间戳、资源、动词等等）， 但是不记录请求或者响应的消息体。
- Request- 记录事件的元数据和请求的消息体，但是不记录响应的消息体。 这不适用于非资源类型的请求。
- RequestResponse- 记录事件的元数据，请求和响应的消息体。这不适用于非资源类型的请求。

你可以使用--audit-policy-file 标志将包含策略的文件传递给 kube-apiserver。 如果不设置该标志，则不记录事件。 注意rules字段必须在审计策略文件中提供。没有规则的策略将被视为非法配置。

例如我们需要记录如下资源的创建、删除、更新、访问审计日志, 其中:
- 计算资源包括：Deployment、StatefulSet、CronJob、DaemonSet、Job、Pod。
- 网络资源包括：Service、Ingress。
- 存储资源包括：ConfigMap、Secret、PersistentVolumeClaim。

需要在控制面节点配置如下审计策略文件给 kube-apiserver, 配置文件保存在 `/etc/kubernetes/audit-policy.yaml` 内容如下

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Record events for the specified resources and operations with metadata level
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
    verbs: ["create", "delete", "update", "get", "list", "watch"]
# Record events for the specified resources and operations with requestresponse level
- level: RequestResponse
  resources:
  - group: apps
    resources: ["deployments", "statefulsets", "daemonsets"]
  - group: batch
    resources: ["cronjobs", "jobs"]
  - group: ""
    resources: ["pods", "services", "persistentvolumes", "persistentvolumeclaims"]
  - group: networking.k8s.io
    resources: ["ingresses"]
  verbs: ["create", "delete", "update", "get", "list", "watch"]
# Don't record events for other resources and operations
- level: None
  resources:
  - group: ""
    resources: ["*"]
  verbs: ["*"]
```

**注意:** 这里吧 Secrets，ConfigMaps 的日志等级设为 Metadata 方式泄露敏感信息, Metadata 级别只记录请求的元数据（请求的用户、时间戳、资源、动词等等），不记录请求或者响应的消息体.

## 审计后端

审计后端实现将审计事件导出到外部存储。Kube-apiserver 默认提供两个后端：

- Log 后端，将事件写入到文件系统
- Webhook 后端，将事件发送到外部 HTTP API

这里我们使用 webhook 作为审计后端, 在 k8s 中部署我们自己的服务 `audit-backend-webhook` 作为审计后端提供服务收集审计日志, 并存储到 elasticsearch. 

## kube-apiserver 配置

### kubernetes Pod 方式部署的 kube-apiserver

如果是 kubernetes Pod 方式部署的 kube-apiserver, 首先, 需要在控制面节点创建审计后端 webhook 配置文件  `/etc/kubernetes/audit-webhook-config.yaml`, 
指定 `audit-backend-webhook` 服务的接口地址(例如 `/reportAuditEvent`)作为审计日志 webhook 后端, 设置
`insecure-skip-tls-verify: true` 跳过证书验证, 文件内容如下:

```yaml
apiVersion: v1
kind: Config
clusters:
- name: audit-webhook
  cluster:
    server: https://cnsp-cls-agent.default.svc.cluster.local/reportAuditEvent
    insecure-skip-tls-verify: true
contexts:
- context:
    cluster: audit-webhook
    user: ""
  name: default-context
current-context: default-context
preferences: {}
users: []
```

其次, 创建 `/etc/kubernetes/audit-policy.yaml`, 添加配置如上 `audit-policy.yaml` 所示.

然后, 修改控制面节点的 kube-apiserver 配置文件 `/etc/kubernetes/manifests/kube-apiserver.yaml`, 添加 volumes 和 volumeMounts 挂载节点, 并在 kube-apiserver 的启动命令后追加如下参数.
配置文件挂载如下

```yaml
volumeMounts:
    - name: audit-policy
      mountPath: /etc/kubernetes/audit-policy.yaml
      readOnly: true
    - name: audit-webhook
      mountPath: /etc/kubernetes/audit-webhook-config.yaml
      readOnly: true
volumes:
  - name: audit-policy
    hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: FileOrCreate
  - name: audit-webhook
    hostPath:
      path: /etc/kubernetes/audit-webhook-config.yaml
      type: FileOrCreate
```

启动命令参数

```yaml
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-webhook-config-file=/etc/kubernetes/audit-webhook-config.yaml
```

另外还需要在 kube-apiserver 的配置文件添加以下配置, 使其能通过集群网络访问 cnsp-cls-agent 服务

```
 dnsPolicy: ClusterFirstWithHostNet
 hostNetwork: true
```

退出并保存  kube-apiserver 配置文件 `/etc/kubernetes/manifests/kube-apiserver.yaml` 后, k8s 会自动重启新的 kube-apiserver 并应用新的配置, 
audit-policy 定义的审计日志即会发送给 `audit-backend-webhook` 的 reportAuditEvent 服务

### k3s 进程作为  kube-apiserver 服务

如果是 k3s 进程直接提供 kube-apiserver 服务的方式, 创建相同的 `/etc/k3s/audit-webhook-config.yaml`, `/etc/k3s/audit-policy.yaml` 文件
然后修改 `/etc/systemd/system/k3s.service` 文件, 在 `ExecStart=/usr/local/bin/k3s` 部分添加如下行

```
'--kube-apiserver-arg' \
'audit-policy-file=/etc/k3s/audit-policy.yaml' \
'--kube-apiserver-arg' \
'audit-webhook-config-file=/etc/k3s/audit-webhook-config.yaml'
```

获取 service 地址

```shell
# kubectl get svc -n jdcloud-sec cnsp-cls-agent
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)           AGE
cnsp-cls-agent   ClusterIP   10.43.148.64   <none>        80/TCP            7m37s
```

添加本地 hosts `/etc/hosts`, 使 k3s 进程能够访问访问 `audit-backend-webhook` 服务

```shell
echo "10.43.148.64  cnsp-cls-agent.default.svc.cluster.local" >> /etc/hosts
```

然后重启 k8s service

```shell
systemctl daemon-reload
systemctl restart k3s.service
```
