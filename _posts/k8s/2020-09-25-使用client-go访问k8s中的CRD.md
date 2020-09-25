---
layout: post 
title: 使用client-go访问k8s中的CRD
categories: golang
description: some word here
keywords: k8s, controller, client-go
---


## 1. 写在前面

> 微信公众号：**[double12gzh]**
> 
> 个人主页: https://gzh.readthedocs.io
> 
> 关注容器技术、关注`Kubernetes`。问题或建议，请公众号留言。

Kubernetes架构的设计模式，我们可以很方便的使用[CRD(Custom Resource Definitions)](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)对k8s API进行扩展。但是问题，通过[client-go](https://github.com/kubernetes/client-go)来获取这些CRD或开发用户自定义控制器，那是比较麻烦的一件事情，除此之外，市面上对于`client-go`的介绍并不是很多。

本文将会通过一个示例，简单介绍一下如何通过client-go获取CRD。

## 2. 写作动机

我在PaaS平台的日常开发工作中，想要将第三方存储厂商集成到Kubernetes集群中时，遇到了这个挑战。计划是使用自定义资源定义来定义诸如文件系统池和文件系统。然后，一个自定义的Operator可以监听这些资源的创建和删除，并负责这些资源的生命周期的管理。

## 3. 定义CR(Custom Resource)

在本文中，我们将以一个简单的例子来进行演示。使用kubectl可以很容易地创建自定义资源定义，对于这个例子，我们将从一个简单的资源定义开始做起：

```yaml
apiVersion: "apiextensions.k8s.io/v1beta1"
kind: "CustomResourceDefinition"
metadata:
  name: "projects.examples-gzh.com"
spec:
  group: "examples-gzh.com"
  version: "v1alpha1"
  scope: "Namespaced"
  names:
    plural: "projects"
    singular: "project"
    kind: "Project"
  validation:
    openAPIV3Schema:
      required: ["spec"]
      properties:
        replicas:
          type: "integer"
          minimum: 1
```

- 确定`Group`的名字。

    在定义CRD时，我们首先需要定义它所在的`Group`(在上面的代码中，其`Group`为：`examples-gzh.com`)。对于Group的定义，为了避免命名冲突，通常会使用一些比较特别的字符串(如：你的个人主页的地址、你公司的域名等)，`Group`名字确定了之后，由于CRD的名字是按`<plural-resource-name>.<api-group-name>`这个格式进行命名的，所以这里我们的CRD的名字为`projects.example-gzh.com`。

- 确定`version`。

    这里的`version`，即`spec.version`。如果你的代码没有开发完成，或者还在快速迭代中，那么，建议你使用`alpha`这样的命名规则。这样的好处是，如果别人想使用你的代码去使用，那么，他单从版本号上就可以很方便的快速知道，你这是一个不稳定的版本。

- schema校验

    在上面我们的CRD中，我们引入了`spec.validation.openAPIV3Schema`，它的作用是对其中的字段进行校验，如果用户在使用我们的CRD时，提供了一个不符合要求的字段后，validation可以很方便的对其进行校验。除了在这里引用validation之外，我们还可以选择在admintion controller中通过`Validate`阶段进行验证，不过样是需要开启admission webhook的。

将上面的代码保存到一个文件中之后，我们就可以通过`kubectl apply -f demo.yaml`进行部署了。我在本机通过minikue启动了一个K8S集群：

```bash
PS C:\Users\guanzenghui> kubectl get po -A
NAMESPACE              NAME                                        READY   STATUS    RESTARTS   AGE
kube-system            coredns-f9fd979d6-7h2b7                     1/1     Running   1          9h
kube-system            etcd-minikube                               0/1     Running   2          9h
kube-system            kube-apiserver-minikube                     1/1     Running   2          9h
kube-system            kube-controller-manager-minikube            0/1     Running   2          9h
kube-system            kube-proxy-p8zb7                            1/1     Running   1          9h
kube-system            kube-scheduler-minikube                     0/1     Running   2          9h
kube-system            storage-provisioner                         1/1     Running   1          9h
kubernetes-dashboard   dashboard-metrics-scraper-c95fcf479-gvhpd   1/1     Running   1          9h
kubernetes-dashboard   kubernetes-dashboard-5c448bc4bf-lpwqh       1/1     Running   1          9h

PS C:\Users\guanzenghui> kubectl version
Client Version: version.Info{Major:"1", Minor:"16+", GitVersion:"v1.16.6-beta.0", GitCommit:"e7f962ba86f4ce7033828210ca3556393c377bcc", GitTreeState:"clean", BuildDate:"2020-01-15T08:26:26Z", GoVersion:"go1.13.5", Compiler:"gc", Platform:"windows/amd64"}
Server Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.2", GitCommit:"f5743093fd1c663cb0cbc89748f730662345d44d", GitTreeState:"clean", BuildDate:"2020-09-16T13:32:58Z", GoVersion:"go1.15", Compiler:"gc", Platform:"linux/amd64"}
```

部署我们的CRD:

```bash
PS C:\Users\guanzenghui\Documents> kubectl apply -f .\Untitled-2.yaml
customresourcedefinition.apiextensions.k8s.io/projects.examples-gzh.com created

PS C:\Users\guanzenghui\Documents> kubectl get crd
NAME                        CREATED AT
projects.examples-gzh.com   2020-09-25T10:40:01Z
```

如果需要查看其详情，可以使用命令: `kubectl describe crd projects.examples-gzh.com`

既然CRD已经创建完成了，接下来我们看一下如何使用这个CRD来创建与之相对应的CR。CR相关的文件内容如下：

```yaml
apiVersion: "examples-gzh.com/v1alpha1"
kind: Project
metadata:
  name: gzh-cr
  namespace: default
spec:
  replica: 2
```

创建CR

```bash
PS C:\Users\guanzenghui\Documents> kubectl apply -f cr.yaml
project.examples-gzh.com/gzh-cr created

PS C:\Users\guanzenghui\Documents> kubectl get Project
NAME     AGE
gzh-cr   39s
```

接下来，我们将使用client-go来获取这个CR。

## 4. 创建golang client

在进行本节前，我假设您已经对client-go、k8s控制器机制有所理解，并且有一定的GoLang的开发经验。

另外，与其它一些讲解Operator的文章不同的是，这些使用CRD的文档会假设你正在使用某种代码生成器来自动生成客户端库。然而，对于这个过程的文档很少，而且从阅读Github上的一些激烈的讨论中，我们可以看出，它仍然是一个正在进行中的工作。

本文中，我将坚持使用（大部分）手动实现的客户端的方式给大家展示。

首先，您可以创建一个自己的项目路径，并安装依赖:

```bash
mkdir github.com/double12gzh/k8s-crd-demo
go get k8s.io/client-go@v0.17.0
go get k8s.io/apimachinery@v0.17.0
```



### 4.1 

## 5. 总结
