---
title: "tcpdump 查看 http 请求内容"
date: 2023-03-06T12:00:00+08:00
isCJKLanguage: true
Description: "tcpdump 查看 http 请求内容的几个常用命令"
Tags: ["Go", "Effective Go"]
Categories: ["Go"]
---

tcpdump 查看 http 请求内容的几个常用命令

- GET
```shell
tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```
- POST
```shell
tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'
```
- PUT
```shell
tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x50555420'
```
- PATCH
```shell
sudo tcpdump -s 0 -A 'tcp dst port 8080 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x50415443'
```
