---
title: "学习 Linux 文件目录之 /etc/resolv.conf"
date: 2023-09-12T12:00:00+08:00
isCJKLanguage: true
Description: "/etc/resolv.conf 文件结构和配置解析"
Tags: ["linux", "dns"]
Categories: ["dns"]
DisableComments: false
---


`/etc/resolv.conf` 文件是Linux和Unix系统中DNS客户端的重要配置文件，它定义了DNS解析器的行为。这个文件由系统管理员手动或者由某些网络服务（如DHCP）自动创建和更新。

这个文件的结构相对简单，主要包括以下几种配置：

1. **nameserver**：定义了用于解析域名的DNS服务器的IP地址。可以有多个`nameserver`条目，每一行定义一个，系统会按照顺序尝试使用。不过，Linux 最多只会使用前三个。例如：
   ```
   nameserver 8.8.8.8
   nameserver 8.8.4.4
   ```
   在这个例子中，系统首先会尝试使用8.8.8.8作为DNS服务器，如果失败，那么会尝试使用8.8.4.4。

2. **search**：指定在进行不完全域名解析时的搜索列表。例如，如果你设置了 search example.com，那么当你尝试解析 host 时，解析器实际上会尝试解析 host.example.com。你可以指定多个搜索域，它们会按照指定的顺序进行尝试。例如：
   ```
   search example.com example.org
   ```
   在这个例子中，如果你尝试解析一个主机名，系统会首先尝试在该主机名后面添加`.example.com`，如果失败，那么会尝试添加`.example.org`。

   在 glibc 2.25 及更早版本中，搜索列表仅限于 6 个域，总共 256 个字符。从 glibc 2.26 开始，搜索列表是无限的。
   ```shell
   $ ldd --version
   ldd (Debian GLIBC 2.31-13+deb11u5) 2.31
   Copyright (C) 2020 Free Software Foundation, Inc.
   This is free software; see the source for copying conditions.  There is NO
   warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
   Written by Roland McGrath and Ulrich Drepper.
   ```

4. **domain**：类似于 search，但只允许指定一个搜索域。如果同时指定了 search 和 domain，那么 domain 会被忽略。例如：
   ```
   domain example.com
   ```
   在这个例子中，当你尝试解析一个没有完整域名的主机名时，系统会自动在该主机名后面添加`.example.com`。
   search 和 domain 的主要区别在于 search 允许你指定多个搜索域，而 domain 只允许你指定一个。在大多数情况下，你应该使用 search，因为它提供了更大的灵活性。 domain 的存在主要是为了向后兼容。在早期的 DNS 客户端配置中，只有 domain 选项。当需要指定多个搜索域时，引入了 search 选项。为了不破坏依赖 domain 的旧系统，domain 选项被保留了下来。因此，虽然在大多数情况下，search 选项可以完全替代 domain. 

5. **options**：定义了一些解析选项。例如：
   ```
   options attempts:2
   ```
   在这个例子中，`attempts:2`表示系统在放弃前应该尝试解析名字的次数。

注意：`/etc/resolv.conf` 文件可能会被网络服务（如DHCP或者NetworkManager）自动更新，如果你想手动管理这个文件，可能需要关闭这些服务的自动更新功能。
