---
title: "k8s资源名称规范"
date: 2022-07-31T10:00:00+08:00
isCJKLanguage: true
Description: "kubernetes 资源名称规范及其正则表达式"
Tags: ["kubernetes", "resource name spec", "资源名称规范"]
Categories: ["k8s"]
---

# 汇总表格

| 类型       | 中文名称 | 英文名称           | 最大长度           | 正则表达式/备注                                             |
| --------------- | -------- | ------------------ | ------------------ | ------------------------------------------------------------ |
| DNS子域名       | Pod      | Pod                | 253                | `^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$` |
|                 |          | ConfigMap          | 253                | 同上                                                         |
|                 | 网络策略 | NetworkPolicy      | 253                | 同上                                                         |
| RFC 1123 标签名 | 服务     | Service            | 63                 |                                                              |
|                 | 命名空间 | Namespace          | 63                 |                                                              |
| RFC 1035 标签名 |          |                    | 63                 |                                                              |
| 路径分段名称    |          | Role               |                    |                                                              |
|                 |          | RoleBinding        |                    |                                                              |
|                 |          | ClusterRole        |                    |                                                              |
|                 |          | ClusterRoleBinding |                    |                                                              |
| helm 应用名称   | 应用名称 |                    | 53                 | ```^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$``` |
| 其他  | 注解键   | Annotations Key    | 317               |长度(253+1+63), 值无限制                                                     |
|         | 注解前缀 | Annotations Prefix | 253                | 正则同 `DNS子域名`, 值无限制                                       |
|         | 注解名称 | Annotations Name | 63                 | 正则同`标签名称`, 值无限制                                         |
|                 | 标签     | Label              | 381              |长度(253+1+63+1+63)                                                 |
|                 | 标签键   | Label Key          | 253                | 正则同 `DNS子域名`                                             |
|                 | 标签名称 | Label Name         | 63                 | ```^([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9]$```           |
|                 | 标签值   | Label Value        | 63                 | 同`标签名称`                                                 |
|                 | 端口名称 | Port Name          | 15                 |                                                              |



# [对象名称和 IDs | Kubernetes](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names/#names)

以下是比较常见的四种资源命名约束。

## DNS 子域名【长度253】

正则表达式：`^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$`

很多资源类型需要可以用作 DNS 子域名的名称。 DNS 子域名的定义可参见 [RFC 1123](https://tools.ietf.org/html/rfc1123
)。 这一要求意味着名称必须满足如下规则：
- 不能超过 253 个字符
- 只能包含小写字母、数字，以及 '-' 和 '.'
- 必须以字母数字开头
- 必须以字母数字结尾
- DNS子域名多级使用 “.” 连接，每一级以字母数字开头，以字母数字结尾，“-”只能出现在每一级的中间

相关资源类型：
- Pod名称
参考自 https://kubernetes.io/zh/docs/concepts/workloads/pods/ 当你为 Pod 对象创建清单时，要确保所指定的 Pod 名称是合法的 [DNS 子域名](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。
- ConfigMap
- 网络策略名称

## RFC 1123 标签名 【长度63】
某些资源类型需要其名称遵循 [RFC 1123](https://tools.ietf.org/html/rfc1123
) 所定义的 DNS 标签标准。也就是命名必须满足如下规则：
- 最多 63 个字符
- 只能包含小写字母、数字，以及 '-'
- 必须以字母数字开头
- 必须以字母数字结尾

相关资源类型：
- 服务名称
- 命名空间名称

## RFC 1035 标签名【长度63】
某些资源类型需要其名称遵循 [RFC 1035](https://tools.ietf.org/html/rfc1035) 所定义的 DNS 标签标准。也就是命名必须满足如下规则：
- 最多 63 个字符
- 只能包含小写字母、数字，以及 '-'
- 必须以字母开头
- 必须以字母数字结尾

相关资源类型：

## 路径分段名称（Path Segment Names）
某些资源类型要求名称能被安全地用作路径中的片段。 换句话说，其名称不能是 `.`、`..`，也不可以包含 `/` 或 `%` 这些字符。

相关资源类型：
Role、ClusterRole、RoleBinding、ClusterRoleBinding

## 其他

### 镜像名称

参考 https://kubernetes.io/zh/docs/concepts/containers/images/#image-names

### 标签【label】【最长381=253+1+63+1+63】

参考 [标签和选择算符 | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)

标签合法格式:[前缀/]标签名称:[标签值]

- 前缀【正则表达式同 DNS子域名】是可选的。如果指定，前缀必须是 DNS 子域：由点（.）分隔的一系列 DNS 标签，总共不超过 253 个字符， 后跟斜杠（/）

- 名称【正则：```^([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9]$```】是必需的，必须小于等于 63 个字符，以字母数字字符（[a-z0-9A-Z]）开头和结尾， 带有破折号（-），下划线（_），点（ .）和之间的字母数字。

有效标签值：【正则表达式同标签名称】

- 必须为 63 个字符或更少（可以为空）
- 除非标签值为空，必须以字母数字字符（[a-z0-9A-Z]）开头和结尾
- 包含破折号（-）、下划线（_）、点（.）和字母或数字

### 应用名称【Release.name】【长度53】

长度限制为 53 个字符，正则表达式为：

```
^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$
```

**提示:** 由于DNS系统的限制，name:字段长度限制为63个字符。因此发布名称限制为53个字符。 Kubernetes 1.3及更早版本限制为24个字符 (名称长度是14个字符)。

**提示：** 向最终用户对象添加标签的自动系统组件（例如 kube-scheduler、kube-controller-manager、 kube-apiserver、kubectl 或其他第三方自动化工具）必须指定前缀。

### 注解【Annotations】【key最长317=253+1+63，value无限制】

参考： 注解 | Kubernetes 语法和字符集

注解合法格式:[前缀/]标签名称:[标签值]

- 前缀是可选的。如果指定，前缀必须是 DNS 子域：由点（.）分隔的一系列 DNS 标签，总共不超过 253 个字符， 后跟斜杠（/）
- 名称是必需的，必须小于等于 63 个字符，以字母数字字符（[a-z0-9A-Z]）开头和结尾， 带有破折号（-），下划线（_），点（ .）和之间的字母数字。

**提示：** 由系统组件添加的注解 （例如，kube-scheduler，kube-controller-manager，kube-apiserver，kubectl 或其他第三方组件），必须为终端用户添加注解前缀。

### 端口名称【最长15】

- 最多 15 个字符
- 只能包含小写字母、数字，以及 '-'，且'-'不能连续
- 必须以字母数字开头
- 必须以字母数字结尾
