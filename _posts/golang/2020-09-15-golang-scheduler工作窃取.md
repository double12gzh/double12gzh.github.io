---
layout: post 
title: 调度器工作窃取
categories: [golang]
description: scheduler
keywords: golang,scheduler
---


**目录**

* TOC
{:toc}

![pic-0](https://gitee.com/double12gzh/wiki-pictures/raw/master/work-stealing/pic-0.png)

## 写在前面

> 本文基于GoLang 1.13

在Go中创建goroutine很容易，也很快速。然而，Go最多只能在每个内核上同时运行一个，并且需要一种方法来存放其他goroutine，并确保负载在各处理器上得到很好的平衡。

## Goroutine队列

Go通过`本地队列`和`全局队列`在两个层面管理处于等待状态的goroutine。`局部队列`与每个处理器相连，而`全局队列`是唯一的，可以被所有处理器使用。

![pic-1](https://gitee.com/double12gzh/wiki-pictures/raw/master/work-stealing/pic-1.png)

每个本地队列的最大容量为256个，之后任何新进入的goroutine都会被推送到全局队列中。下面这个例子中的程序，可以产生成千上万的goroutine。

```golang
package main

import (
    "runtime"
	"sync"
)

const COUNT = 3000

func main() {
    runtime.GOMAXPROCS(2)

	var wg sync.WaitGroup

	wg.Add(COUNT)

	for i := 0; i < COUNT; i++ {
		go func() {
			a := 0

			for i := 0; i < 1e7; i++ {
				a++
			}

			wg.Done()

		}()
	}

	wg.Wait()
}
```

我的电脑上有4个处理器，但是我在程序中设置了只用其中2个，以下是go scheduler做调度的trace记录。

```bash
Lenovo@Lenovo-PC MINGW64 /f/hello
$ GODEBUG=schedtrace=1000 go run main.go
```

![pic-011](https://gitee.com/double12gzh/wiki-pictures/raw/master/work-stealing/pic-011.png)

上述trace显示了全局队列中的goroutine数量，括号[xx yy]中为runqueue和本地队列（分别为P0和P1）。当本地队列满了，达到256个等待的goroutine时，接下来的goroutine将堆叠在全局队列中，我们可以看到runqueue属性在增长

Goroutine并不是只有在本地队列满了的时候才会进入全局队列，当Go向调度器注入一个goroutine列表时（例如来自网络轮询器或垃圾收集过程中处于sleep状态的goroutine），它也会被推送进去。

下面是前面例子的图。

![pic-2](https://gitee.com/double12gzh/wiki-pictures/raw/master/work-stealing/pic-2.png)

然而，您也许想知道为什么在前面的例子中，P0的本地队列不是空的。Go使用不同的策略来确保每个处理器保持忙碌。

## 工作窃取

当一个处理器没有任何工作（即G）时，它应用以下规则，直到可以有一个可以执行的G：

- 从本地队列中抽取G
- 从全局队列中抽取G
- 从网络轮循器中抽调G
- 从其它P的本地队列中获取G

在我们前面的例子中，主函数正在P1上运行并创建goroutine，当第一个goroutine在P1的本地队列中添加后，P0正在寻找可以执行的G。然而，它的本地队列、全局队列和网络轮询器都是空的。所以它只能从P1的本地队列中去"窃取"可执行的G:

![pic-3](https://gitee.com/double12gzh/wiki-pictures/raw/master/work-stealing/pic-3.png)

下面的trace显示了在"窃取"前后的一个调度trace:

![pic-012](https://gitee.com/double12gzh/wiki-pictures/raw/master/work-stealing/pic-012.png)

trace显示了一个处理器如何从其他处理器那里窃取G的。它从本地队列中抽取了一半的goroutine；在7个goroutine中，有4个被窃取--其中一个立即在P0上运行，而其他三个则进入本地队列。现在各处理器之间的工作已经很平衡了。这一点可以通过执行跟踪得到证实：

![pic-4](https://gitee.com/double12gzh/wiki-pictures/raw/master/work-stealing/pic-4.png)

goroutine很好调度，由于没有I/O，所以goroutine是链式的，没有切换。现在我们来看看当涉及到一些I/O如文件操作时会发生什么。

## I/O和全局队列

```golang
package main

import (
	"io/ioutil"
	"strconv"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func() {
			a := 0

			for i := 0; i < 1e6; i++ {
				a += 1
				if i == 1e6/2 {
					bytes, _ := ioutil.ReadFile(`add.txt`)
					inc, _ := strconv.Atoi(string(bytes))
					a += inc
				}
			}

			wg.Done()
		}()
	}

	wg.Wait()
}
```

变量`a`现在会根据写入文件中的数字时时地递增。以下是新的trace:

![pic-5](https://gitee.com/double12gzh/wiki-pictures/raw/master/work-stealing/pic-5.png)

通过这个例子，我们可以看到每个goroutine并不是只由一个处理器处理的。在系统调用的情况下，当调用完成后，Go会使用网络轮询器，把全局队列中的goroutine拉回来。

下面是对G35一个说明：

![pic-6](https://gitee.com/double12gzh/wiki-pictures/raw/master/work-stealing/pic-6.png)

由于一个处理器在没有任务时可以从全局队列中提取G，所以第一个可用的P将运行goroutine。这种行为解释了为什么一个goroutine会在不同的P上运行，并展示了Go如何优化系统调用与让其他goroutine在资源空闲时运行。
