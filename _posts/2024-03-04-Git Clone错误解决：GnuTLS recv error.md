---
title: Git Clone错误解决：GnuTLS recv error (-110): The TLS connection was non-properly terminated.
tags: git
aside:
  toc: true
---

Git Clone 错误解决：GnuTLS recv error (-110): The TLS connection was non-properly terminated.

<!--more-->

## 1. 问题描述

在使用 `git clone` 命令时，出现如下错误：

```shell
fatal: unable to access ...: GnuTLS recv error (-110): The TLS connection was non-properly terminated.
```

## 2. 解决方法

```shell
apt install apt-transport-https
```

apt-transport-https 是一个用于在 Ubuntu 和 Debian 系统上使用 HTTPS 协议进行软件包管理的工具。它允许用户通过 HTTPS 协议从安全的源中下载软件包，以确保软件包的完整性和安全性。
