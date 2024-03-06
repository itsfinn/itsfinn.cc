---
title: "Kubernetes 基于 DNS 的服务发现"
date: 2024-03-09T12:00:00+08:00
isCJKLanguage: true
Description: "Kubernetes 基于 DNS 的服务发现(kube-dns 规约)"
Tags: ["envoy", "introduction", "介绍"]
Categories: ["envoy"]
DisableComments: false
---

原文地址: https://github.com/kubernetes/dns/blob/master/docs/specification.md

Kubernetes DNS 项目的规范文档
<!--more-->

# 0 - 关于本文档

本文档是基于 DNS 的 Kubernetes 服务发现的规范。
虽然 Kubernetes 中的服务发现可以通过其他协议和机制提供，
但 DNS 非常常用，是一个强烈推荐的附加组件。

实际的 DNS 服务本身不需要由默认的 Kube-DNS 实现提供。
本文档旨在为实现之间的通用性提供一个基准。

# 1 - 架构版本

本文档介绍架构的 1.1.0 版。

# 2 - 资源记录

任何基于 DNS 的 Kubernetes 服务发现解决方案都必须提供下面所述的资源记录 （RR） 才能被视为符合此规范。

## 2.1 - 定义

在下面的 RR 说明中，不在尖括号中的值 `< >` 是文本。尖括号中值的含义在下面或特定记录的描述中定义。

- `<zone>` = 配置的集群域，例如 cluster.local
- `<ns>` = 命名空间
- `<ttl>` = 记录的标准 DNS 生存时间值

在下面的 RR 说明中，斜体字应使用以下定义。

*hostname*

- 按优先级顺序，endpoint 的 *hostname* 为：
  - endpoint 的 `hostname` 字段值
  - endpoint 的唯一的系统分配的标识符。本规范不规定此标识符的确切格式和来源。
    但是，必须能够使用它来标识服务上下文中的特定 endpoint。

    这在未定义显式 endpoint 主机名的情况下使用

*ready*

- 如果 endpoint 位于 EndpointSubset 对象的 `addresses` 字段中，
  或者相应的服务将 `service.alpha.kubernetes.io/tolerate-unready-endpoints` annotation 设置为 `true` ，
  则认为该 endpoint 已就绪。

查询数据和 Kubernetes 中的数据之间的所有比较都不区分大小写。

## 2.2 - 架构版本记录

必须有一个名为 `dns-version.<zone>.` 的 `TXT` 记录，其中包含此群集中使用的 DNS 架构的语义版本。

- 记录格式：
  - `dns-version.<zone>. <ttl> IN TXT <schema-version>`

- 问题示例：
  - `dns-version.cluster.local. IN TXT`

- 答案示例：
  - `dns-version.cluster.local. 28800 IN TXT "1.1.0"`

> **验证一下**
>
> ```shell
> $ kubectl apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
> pod/dnsutils created
> $ kubectl get pods
> NAME       READY   STATUS   RESTARTS   AGE
> dnsutils   1/1     Runing   0          31s
> $ kubectl exec -it dnsutils -- dig +nocomments +noquestion +noauthority +noadditional +nostats -t TXT dns-version.cluster.local
>
> ; <<>> DiG 9.11.6-P1 <<>> +nocomments +noquestion +noauthority +noadditional +nostats -t TXT dns-version.cluster.local
> ;; global options: +cmd
> dns-version.cluster.local. 30   IN      TXT     "1.1.0"
> ```
