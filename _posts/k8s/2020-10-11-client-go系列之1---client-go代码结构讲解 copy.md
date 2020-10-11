---
layout: post 
title: client-go系列之1---client-go代码结构讲解
categories: k8s
description: some word here
keywords: k8s, client-go, restclient, clientset, dynamicclient, discoveryclient
excerpt: 摘要：分析client-go的代码结构。
---


## 1. 写在前面

本系列内容都是基于这个版本的[client-go](https://github.com/kubernetes/client-go/tree/becbabb360023e1825a48b4db85f454e452ae249)进行讲解，不
同版本的略有差异。

```bash
[root@77DDE94FF07FCC1-wsl /ACode/client-go] git rev-parse HEAD
becbabb360023e1825a48b4db85f454e452ae249
```

## 2. 代码结构

从这个包的名字可以很明显的知道，`client-go`其实主要是提供了用户与k8s交互时使用客户端，方便大家编程。


> 个别目录已过滤掉。

```bash
[root@77DDE94FF07FCC1-wsl /ACode/client-go] tree -d -L 1 -I "testing|examples|*_test*|Godeps|third_party|metadata|deprecated|restmapper"
.
├── discovery                   # 定义DsicoveryClient客户端。作用是用于发现k8s所支持GVR(Group, Version, Resources)。
├── dynamic                     # 定义DynamicClient客户端。可以用于访问k8s Resources(如: Pod, Deploy...)，也可以访问用户自定义资源(即: CRD)。
├── informers                   # k8s中各种Resources的Informer机制的实现。
├── kubernetes                  # 定义ClientSet客户端。它只能用于访问k8s Resources。每一种资源(如: Pod等)都可以看成是一个客端，而ClientSet是多个客户端的集合，它对RestClient进行了封装，引入了对Resources和Version的管理。通常来说ClientSet是client-gen来自动生成的。
├── listers                     # 提供对Resources的获取功能。对于Get()和List()而言，listers提供给二者的数据都是从缓存中读取的。
├── pkg                         
├── plugin                      # 提供第三方插件。如：GCP, OpenStack等。
├── rest                        # 定义RestClient，实现了Restful的API。
├── scale                       # 定义ScalClient。用于Deploy, RS, RC等的扩/缩容。
├── tools                       # 定义诸如SharedInformer、Reflector、DealtFIFO和Indexer等常用工具。实现client查询和缓存机制，减少client与api-server请求次数，减少api-server的压力。
├── transport
└── util                        # 提供诸如WorkQueue、Certificate等常用方法。

12 directories
```

client-go中定义的比较重要的client有： RestClient, ClientSet, DiscoveryClient和DynamicClient。其中，RestClient是所有客户端的基础，后三者都是对RestClient的封装。RestClient它通过kubeconfig与k8s-api-server进行交互。详细结构如下图：

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-10-11-client-go/0-client-go-arch.png)

