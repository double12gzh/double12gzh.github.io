---
layout: post 
title: client-go系列之2---管理kubeconfig
categories: k8s
description: some word here
keywords: k8s, client-go, restclient, clientset, dynamicclient, discoveryclient
excerpt: 摘要：分析client-go如何管理kubeconfig。
---


## 1. 写在前面

## 2. 集群配置管理

k8s的配置文件默认会存放在~/.kube/config中，其内容如下：

```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0xxxxxxxxCg==
    server: https://127.0.0.1:33629
  name: kind-kind
contexts:
- context:
    cluster: kind-kind
    user: kind-kind
  name: kind-kind
current-context: kind-kind
kind: Config
preferences: {}
users:
- name: kind-kind
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1Jxxxxxxx==
    client-key-data: LS0tLS1yyyyyyyCg==
```

上面这是单个配置文件情况，client-go提供了合并多个配置文件的能力，合并完成后如下：

```bash
apiVersion: v1

clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQtCg==
    server: https://127.0.0.1:51107
  name: kind-guan
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJtCg==
    server: https://127.0.0.1:33629
  name: kind-kind

contexts:
- context:
    cluster: kind-guan
    user: kind-guan
  name: kind-guan
- context:
    cluster: kind-kind
    user: kind-kind
  name: kind-kind


current-context: kind-kind

kind: Config

preferences: {}

users:
- name: kind-guan
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUUtLS0tLQo=
    client-key-data: LS0tLS1CRUdJTiBxxxxxS0tCg==
- name: kind-kind
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJxxxxxxEUtLS0tLQo=
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSxxxxg==
```

client-go中对于配置文件的管理可以简单概括为以下两点：

* 加载配置文件
* 合并配置文件

接下来分别从代码层面看一下。

### 2.1 加载配置文件

### 2.2 合并配置文件