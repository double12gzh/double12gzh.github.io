---
layout: post 
title: Go是如何回收goroutine的
categories: [golang]
description: golang goroutine
keywords: golang,goroutine
---


**目录**

* TOC
{:toc}

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/go%E6%98%AF%E5%A6%82%E4%BD%95%E5%9B%9E%E6%94%B6goroutine%E7%9A%84/pic-0.png)

## 1. 写在前面

> 微信公众号：**[double12gzh]**
> 
> 关注容器技术、关注`Kubernetes`。问题或建议，请公众号留言。

> 本文是基于golang 1.13

Goroutines易于创建，堆栈小，上下文切换快。由于这些原因，开发人员喜欢它们，并经常使用它们。然而，一个程序如果产生许多这样生命周期很短的goroutine，那将会花费相当多的时间来创建和销毁它们。

## 2. 生命周期

> 下面我们将以一个简单的例子来看一下，golang中是如何重用goroutine的。这个例子中的代码主要功能是实现如何计算质数

```golang
package main

// 发送一个序列2, 3, 4, ... 到channel 'ch'.
func Generate(ch chan<- int) {
	for i := 2; ; i++ {
		ch <- i // Send 'i' to channel 'ch'.
	}
}

// 把channel 'in'中的数据复制到channel 'out'。
// 删除那些可以被'prime'整除的数据。
func Filter(in <-chan int, out chan<- int, prime int) {
	for {
		i := <-in // 从channel 'in'中获取数据。
		if i%prime != 0 {
			out <- i // 把'i'发送到channel 'out'中。
		}
	}
}

// The prime sieve: Daisy-chain Filter processes.
func main() {
	ch := make(chan int) // 创建一个新的channel。
	go Generate(ch)      // 启动goroutine.
	for i := 0; i < 10; i++ {
		prime := <-ch
		print(prime, "\n")
		ch1 := make(chan int)
		go Filter(ch, ch1, prime)
		ch = ch1
	}
}
```

数百个goroutine将被用来过滤数字，同时，Go负责管理这些goroutine的创建和销毁。实际上，Go为每个`p`维护着一个空闲的goroutine列表，如下图

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/go%E6%98%AF%E5%A6%82%E4%BD%95%E5%9B%9E%E6%94%B6goroutine%E7%9A%84/pic-1.png)

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/go%E6%98%AF%E5%A6%82%E4%BD%95%E5%9B%9E%E6%94%B6goroutine%E7%9A%84/pic-2.png)

将这个列表保持在本地，每个`P`带来的好处是不需要使用任何锁来推送或获取一个空闲的goroutine。然后，当一个goroutine从当前工作中退出时，它将被推送到这个空闲列表中。

![](https://gitee.com/double12gzh/wiki-pictures/blob/master/go%E6%98%AF%E5%A6%82%E4%BD%95%E5%9B%9E%E6%94%B6goroutine%E7%9A%84/pic-3.png)

然而，为了更好地分配释放的goroutine，调度器也有自己的列表。实际上，它有两个列表：一个是包含有分配栈的goroutine，另一个是保持释放栈的goroutine--这个细节将在下一节解释。如下图：

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/go%E6%98%AF%E5%A6%82%E4%BD%95%E5%9B%9E%E6%94%B6goroutine%E7%9A%84/pic-5.png)

由于任何线程都可以访问中心列表，因此有一个锁保护中心列表。当本地列表长度超过64时，由调度器持有的列表会从`P`获取goroutine。然后，一半的goroutine会被转移到中心列表中。

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/go%E6%98%AF%E5%A6%82%E4%BD%95%E5%9B%9E%E6%94%B6goroutine%E7%9A%84/pic-6.png)

不过，虽然看起来很直接，但这个工作流程对goroutine的内存分配有一些规定。

## 3. 必备条件

回收goroutine是节省其分配成本的好方法。然而，由于堆栈是动态增长的，所以存在的goroutine可能会根据所做的工作最终形成一个大堆栈。出于这个原因，当堆栈已经增长，即超过2K时，Go不会保留堆栈。
在所举的例子中，计算质数是一个相当轻的过程，并不会增长goroutine的栈，让Go重用它们。

下面是一些benchmark测试的结果：

```bash
With recycling               Without recycling
name           time/op       name           time/op
PrimeNumber     12.7s ± 3%   PrimeNumber     12.1s ± 3%
PrimeNumber-8   2.27s ± 4%   PrimeNumber-8   2.13s ± 3%

name           alloc/op      name           alloc/op
PrimeNumber    1.83MB ± 0%   PrimeNumber    5.82MB ± 4%
PrimeNumber-8  1.52MB ± 7%   PrimeNumber-8  5.90MB ±21%
```

我们需要注意的是，没有原生的方式来禁用它，我是直接在Go标准库中禁用该功能。在那个例子中，重用goroutines减少了3个分配!

让我们回顾一下另一种情况，当goroutines使用较大的堆栈时，因此，不适合循环使用。

```golang
func Ping() {
   for i := 0; i < 20; i++ {
      var wg sync.WaitGroup

      for g := 0; g < 20; g++ {
         wg.Add(1)

         go func() {
            if _, err := http.Get(`https://double12gzh.github.io`); err != nil {
                painc(err)
            }

            wg.Done()
         }()

      }

      wg.Wait()
   }
}
```

下面是benchmark的结果：

```bash
With recycling               Without recycling
name           time/op       name           time/op
PingUrl     12.8s ± 2%       PingUrl     12.8s ± 3%
PingUrl-8   12.6s ± 0%       PingUrl-8   12.7s ± 3%

name           alloc/op      name           alloc/op
PingUrl    9.21MB ± 0%       PingUrl    9.44MB ± 0%
PingUrl-8  9.28MB ± 0%       PingUrl-8  9.43MB ± 0%
```

这里的影响是相当低的，因为goroutines得到了一个更大的堆栈。根据你的程序的性质，它可以带来巨大的优势。