---
layout: fragment
title: 容器 namespaces
tags: [docker, namespace]
description: logseq descripition
keywords: docker, namespace, k8s
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

在创建容器时，Docker 和其他容器运行时通常会使用以下几种 Linux namespaces 来隔离容器的运行环境：
- `PID`（Process ID）：PID namespace 用于隔离进程 ID，每个 namespace 有自己的 PID 空间。在 PID namespace 中，一个进程可以有一个在该 namespace 中唯一的 PID，这个 PID 在其他 namespace 中可能代表另一个进程或者根本不存在。
- `NET`（Network）：NET namespace 用于隔离网络接口，每个 namespace 有自己的网络设备、路由表、防火墙规则等。
- `IPC`（Interprocess Communication）：IPC namespace 用于隔离 System V IPC 对象和 POSIX 消息队列，每个 namespace 有自己的 IPC 资源。
- `MNT`（Mount）：MNT namespace 用于隔离文件系统挂载点，每个 namespace 有自己的挂载点，一个 namespace 中的挂载操作不会影响其他 namespace。
- `UTS`（UNIX Time-sharing System）：UTS namespace 用于隔离主机名和域名，每个 namespace 可以有自己的主机名。
- `USER`（User）：USER namespace 用于隔离用户 ID 和组 ID，每个 namespace 可以有自己的用户和组，一个 namespace 中的用户可能在其他 namespace 中不存在或者代表另一个用户。
- `CGROUP`（Control Group）：CGROUP namespace 用于隔离 cgroup 根目录，每个 namespace 可以有自己的 cgroup 层次结构。
‎
这些 namespace 一起为容器提供了一个隔离的运行环境，使得容器内的进程无法直接访问容器外的资源，从而提高了系统的安全性。
