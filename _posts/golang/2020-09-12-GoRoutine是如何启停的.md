---
layout: post 
title: goroutine是如何启停的
categories: [golang]
description: golang
keywords: golang
---


**目录**

* TOC
{:toc}

## 写在前面

> 本文基于GoLang 1.14

在Go中，goroutine不过是一个Go结构，其中包含了关于正在运行的程序的信息，如堆栈、程序计数器或其当前的操作系统线程。Go调度器会处理这些信息，给它们提供运行时间。调度器在goroutine的启动和退出时也要注意，这两个阶段需要小心管理。

## 启动

创建一个goroutine非常的简单，如下：

```golang
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(1)

	go func() {
		fmt.Println("Hello")
		wg.Done()
	}()

	fmt.Println("Done")
	wg.Wait()
}
```

主函数在打印消息之前，会启动一个goroutine。由于goroutine会有自己的运行时间，Go会通知运行时建立一个新的goroutine，也就是说:
- 创建堆栈。
- 收集当前程序计数器或调用者的数据信息。
- 更新goroutine的内部数据，如ID或状态。

但是，goroutine不会立即获得任何运行时间。新创建的goroutine将被放到在本地队列的开头，并在下一轮GoLang调度器中运行。

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-12-GoRoutine%E6%98%AF%E5%A6%82%E4%BD%95%E5%90%AF%E5%81%9C%E7%9A%84/%E5%9B%BE1.png)

把goroutine放在队列的头，目的是使它在当前goroutine之后第一个运行。如果有work-stealing的情况发生，它将在同一线程或另一线程上运行。

我们可以从下面的汇编代码中看到goroutine的创建：

![](https://gitee.com/double12gzh/wiki-pictures/raw/master/2020-09-12-GoRoutine%E6%98%AF%E5%A6%82%E4%BD%95%E5%90%AF%E5%81%9C%E7%9A%84/%E5%9B%BE2.png)

一旦goroutine被创建并推送到本地的goroutine队列中，它就直接进入主函数的下一条指令。

## 退出

当一个goroutine结束时，为了不浪费CPU时间，Go必须调度另一个goroutine。同时，它还会保留当前这个goroutine，以便以后重复使用。

然而，GoLang需要一种方法来感知goroutine的结束。这个控制是在创建goroutine的过程中。

在创建goroutine时，Go在将程序计数器设置为goroutine调用的真正函数之前，会将栈设置为一个名为`goexit`的函数，这样可以保证goroutine在结束工作后调用函数`goexit`。

关于上面的描述，我们通过以下代码进行展示一下：

```golang
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(1)

	go func() {
		var skInt int
		for {
			_, file, line, ok := runtime.Caller(skInt)

			if !ok {
				break
			}

			fmt.Printf("%s:%d\n", file, line)
			skInt++
		}

		wg.Done()
	}()

	fmt.Println("Done")
	wg.Wait()
}
```

运行代码得到如下输出：

```bash
F:\hello>go run main.go
Done
F:/hello/main.go:16
D:/Go/src/runtime/asm_amd64.s:1374
```

从`asm_amd64.s`这个文件中，我们可以看到有如下的函数的定义：

```bash
// The top-most function running on a goroutine
// returns to goexit+PCQuantum.
TEXT runtime·goexit(SB),NOSPLIT,$0-0
	BYTE	$0x90	// NOP
	CALL	runtime·goexit1(SB)	// does not return
	// traceback from goexit1 must hit code range of goexit
	BYTE	$0x90	// NOP
```

我们可以看到，GoLang将会切换到`g0`这个goroutine从而去调度其它的goroutine。

在我们代码中，我们也可以调用`runtime.Goexit()`去主动退出。

```golang
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	var wg sync.WaitGroup

	wg.Add(1)

	go func() {
		defer wg.Done()
		runtime.Goexit()

		fmt.Println("exist")
	}()

	wg.Wait()
}
```

这个函数将首先运行`defer`函数，然后当一个goroutine退出时，将调用之前看到的同一个函数。