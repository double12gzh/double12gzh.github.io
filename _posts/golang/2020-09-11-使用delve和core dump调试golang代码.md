---
layout: post 
title: 使用delve和coredump调试golang代码
categories: [golang]
description: golang,state-machine
keywords: golang
---


**目录**

* TOC
{:toc}


## 写在前面

> 本文基于GoLang delve 1.4.1。

`coredump`是一个包含程序意外终止时的内存快照的文件。它可以用于事后调试，以了解崩溃发生的原因以及其中涉及的变量。通过GOTRACEBACK，Go提供了一个环境变量来控制程序崩溃时产生的输出。这个变量还可以强制生成`coredump`，这样以来就使得调试成为可能。

## GOTRACEBACK

`GOTRACEBACK`可以用于控制程序程序崩溃时输出内容的多少，它可以有以下一些取值：

- `none` 不显示goroutine的堆栈调用的trace
- `single` （默认值）显示当前goroutine的堆栈调用的trace
- `all`显示用户创建的所有的goroutine的堆栈调用的trace
- `system`显示包含运行时goroutine及其它所有goroutine的堆栈调用的trace
- `crash`类似于`system`，不同的是，它同时也会产生coredump

最后一个选项使我们能够在程序崩溃时进行调试。如果你没有得到足够的日志，或者崩溃无法重现，这可能是一个不错的选择。让我们用下面的程序来举个例子:

```golang
package main

import "math/rand"

func main() {
	sum := 0
	for {
		n := rand.Intn(1e6)
		sum += n
		if sum%42 == 0 {
			panic(":(")
		}
	}
}
```

运行上面的程序，我们很快就会发现程序出错了：

```bash
F:\hello>go run main.go
panic: :(

goroutine 1 [running]:
main.main()
        F:/hello/main.go:11 +0x8f
exit status 2
```

但这里的问题是，我们无法从堆栈调用的trace中判断出具体是哪个值引起的崩溃。当然了，您可以通过添加日志的方法去一步步的定位问题具体出现在哪里，但我们并不总是能知道在我们应该把日志添加在哪里。另外呢，当一个问题无法重现时，写测试用例并确保它被修复也是相当困难的。

总结一下上面的思路：在添加日志和运行应用程序之间迭代，直到它崩溃，并查看可能的原因运行后。

是否有其它的办法呢？答案是：有的。

我们可以用GOTRACEBACK=crash再运行一次，这样将有比较详细的输出了，因为我们现在已经打印出了所有的goroutines，包括运行时的。除此之外，也生成了相应的`coredump`文件（注意，coredump文件是由`SIGABRT`触发产生的，win上目前应该还没有支持产生coredump）。

```bash
F:\hello>set GOTRACEBACK=crash

F:\hello>go run main.go
panic: :(

goroutine 1 [running]:
panic(0x469600, 0x48d7b0)
        D:/Go/src/runtime/panic.go:1064 +0x470 fp=0xc00002bf58 sp=0xc00002bea0 pc=0x42fd90
main.main()
        F:/hello/main.go:11 +0x8f fp=0xc00002bf88 sp=0xc00002bf58 pc=0x461d8f
runtime.main()
        D:/Go/src/runtime/proc.go:204 +0x209 fp=0xc00002bfe0 sp=0xc00002bf88 pc=0x432929
runtime.goexit()
        D:/Go/src/runtime/asm_amd64.s:1374 +0x1 fp=0xc00002bfe8 sp=0xc00002bfe0 pc=0x45abc1

goroutine 2 [force gc (idle)]:
runtime.gopark(0x47cf20, 0x4d1b20, 0x1411, 0x1)
        D:/Go/src/runtime/proc.go:306 +0xfa fp=0xc000027fb0 sp=0xc000027f90 pc=0x432cfa
runtime.goparkunlock(...)
        D:/Go/src/runtime/proc.go:312
runtime.forcegchelper()
        D:/Go/src/runtime/proc.go:255 +0xcd fp=0xc000027fe0 sp=0xc000027fb0 pc=0x432b8d
runtime.goexit()
        D:/Go/src/runtime/asm_amd64.s:1374 +0x1 fp=0xc000027fe8 sp=0xc000027fe0 pc=0x45abc1
created by runtime.init.6
        D:/Go/src/runtime/proc.go:243 +0x3c

goroutine 3 [GC sweep wait]:
runtime.gopark(0x47cf20, 0x4d1c00, 0x140c, 0x1)
        D:/Go/src/runtime/proc.go:306 +0xfa fp=0xc000029fa8 sp=0xc000029f88 pc=0x432cfa
runtime.goparkunlock(...)
        D:/Go/src/runtime/proc.go:312
runtime.bgsweep(0xc000018070)
        D:/Go/src/runtime/mgcsweep.go:163 +0xb2 fp=0xc000029fd8 sp=0xc000029fa8 pc=0x41e132
runtime.goexit()
        D:/Go/src/runtime/asm_amd64.s:1374 +0x1 fp=0xc000029fe0 sp=0xc000029fd8 pc=0x45abc1
created by runtime.gcenable
        D:/Go/src/runtime/mgc.go:217 +0x67

goroutine 4 [GC scavenge wait]:
runtime.gopark(0x47cf20, 0x4d1c60, 0x140d, 0x1)
        D:/Go/src/runtime/proc.go:306 +0xfa fp=0xc000037f78 sp=0xc000037f58 pc=0x432cfa
runtime.goparkunlock(...)
        D:/Go/src/runtime/proc.go:312
runtime.bgscavenge(0xc000018070)
        D:/Go/src/runtime/mgcscavenge.go:265 +0xe5 fp=0xc000037fd8 sp=0xc000037f78 pc=0x41c105
runtime.goexit()
        D:/Go/src/runtime/asm_amd64.s:1374 +0x1 fp=0xc000037fe0 sp=0xc000037fd8 pc=0x45abc1
created by runtime.gcenable
        D:/Go/src/runtime/mgc.go:218 +0x89
exit status 2
```

> 对于生成的coredump，我们可以使用[delve](https://github.com/go-delve/delve)或[GDB](https://www.gnu.org/software/gdb/)进行调试。

## Delve

Delve是一个用Go编写的Go程序的调试器。它可以通过在用户代码以及运行时的任何地方添加断点来逐步进行调试，甚至可以使用命令dlv core调试coredump，该命令将二进制和coredump作为参数。

命令运行后，我们就可以开始与coredump进行交互。

