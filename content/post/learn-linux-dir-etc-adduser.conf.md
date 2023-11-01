---
title: "学习 Linux 文件目录之 /etc/adduser.conf"
date: 2023-09-12T12:00:00+08:00
isCJKLanguage: true
Description: "/etc/adduser.conf 文件结构和配置解析"
Tags: ["linux", "adduser"]
Categories: ["linux"]
DisableComments: false
---


`/etc/adduser.conf` 文件是用于配置用户添加过程中的默认行为和参数的配置文件。本文记录了该文件的结构和常见配置项的含义.
<!--more-->


`/etc/adduser.conf` 文件是用于配置用户添加过程中的默认行为和参数的配置文件。以下是该文件的结构和常见配置项的含义：

```shell
# /etc/adduser.conf: `adduser' 配置文件。
# 详细文档请参考 adduser(8) 和 adduser.conf(5)。

# DSHELL 变量指定系统上的默认登录 shell。
DSHELL=/bin/bash

# DHOME 变量指定包含用户主目录的目录。
DHOME=/home

# 如果 GROUPHOMES 为 "yes"，则主目录将创建为 /home/groupname/user。
GROUPHOMES=no

# 如果 LETTERHOMES 为 "yes"，则创建的主目录将有一个额外的
# 目录(用户名的第一个字母)。例如：/home/u/user。
LETTERHOMES=no

# SKEL 变量指定包含 "模板" 用户文件的目录；换句话说，
# 即将在创建新用户时复制到其主目录的文件，如示例的 .profile。
SKEL=/etc/skel

# FIRST_SYSTEM_[GU]ID 到 LAST_SYSTEM_[GU]ID 是分配给
# 动态分配的管理和系统帐户/组的 UID 范围。
# 请注意，系统软件（如 base-passwd 包分配的用户）可能假设 UID 小于 100 未分配。
FIRST_SYSTEM_UID=100
LAST_SYSTEM_UID=999

FIRST_SYSTEM_GID=100
LAST_SYSTEM_GID=999

# FIRST_[GU]ID 到 LAST_[GU]ID 是动态分配的用户帐户/组的 UID 范围。
FIRST_UID=1000
LAST_UID=59999

FIRST_GID=1000
LAST_GID=59999

# USERGROUPS 变量可以是 "yes" 或 "no"。如果是 "yes"，每个创建的用户
# 将被分配一个用作默认组的独立组。如果是 "no"，每个创建的用户将被放置
# 在其 gid 为 USERS_GID 的组中（参见下面）。
USERGROUPS=yes

# 如果 USERGROUPS 是 "no"，那么 USERS_GID 应该是系统上 `users'（或等效组）的 GID。
USERS_GID=100

# 如果设置了 DIR_MODE，目录将以指定的模式创建。否则将使用默认模式 0755。
DIR_MODE=0755

# 如果 SETGID_HOME 是 "yes"，对于具有自己组的用户，主目录将设置 setgid 位。
# 这是 adduser 的版本 << 3.13 的默认设置。由于它有一些不好的副作用，
# 我们不再默认设置。如果您仍然想要它，仍然可以在此处设置。
SETGID_HOME=no

# 如果设置了 QUOTAUSER，则将从该用户为新用户设置默认配额，
# 使用 `edquota -p QUOTAUSER newuser'。
QUOTAUSER=""

# 如果设置了 SKEL_IGNORE_REGEX，则在创建新主目录时，
# adduser 将忽略与该正则表达式匹配的文件。
SKEL_IGNORE_REGEX="dpkg-(old|new|dist|save)"

# 如果要让 --add_extra_groups 选项添加用户到其他组，请设置此项。
# 这是新的非系统用户将被添加到的组的列表
# 默认：
#EXTRA_GROUPS="dialout cdrom floppy audio video plugdev users"

# 如果 ADD_EXTRA_GROUPS 设置为非零值，则上面的 EXTRA_GROUPS 选项
# 将成为添加新的非系统用户的默认行为
#ADD_EXTRA_GROUPS=1


# 还可以根据此正则表达式检查用户和组名。
#NAME_REGEX="^[a-z][-a-z0-9_]*\$"
```

通过修改这些配置项，可以自定义用户添加过程中的默认行为和参数，以适应特定的需求和安全策略。
