---
layout: post 
title: GoLang中的Context
categories: [golang]
description: golang,context
keywords: golang,context
excerpt: 摘要：介绍如何使用GoLang中的Context。
---


**目录**

* TOC
{:toc}


![](https://gitee.com/double12gzh/wiki-pictures/raw/master/Go-How-to-Reduce-Lock-Contention-with-the-Atomic-Package/%E5%9B%BE0.png)

## 1. 背景
我们在开发Golang中的应用时，通常会使用Contexts来控制和管理所依赖的应用中非常重要的数据，例如并发编程中的`cancellation`和`data share`。

在GoLang中，`context`作为context的交互的入口，它被认为GoLang中非常重要一个包。假如当前你还没有遇到与`context`相关的操作，那么，相信在不久的将来也肯定会遇到，它的使用非常的广泛，如果你认真观察过，你会发现，其它许多的包也会依赖于`context`。

> [`context`的官方文档](https://golang.org/pkg/context)

## 2. 正文

### 2.1 带value的context
`context`其中一个比较常用的用法就是**共享数据**，另外，我们也可以使用`request`中携带的数据。

当你有多个方法/函数之间需要共享数据时，你可以尝试使用一下`context`中的方法`WithValue()`，它的用法非常简单：`context.WithValue`。

> 这个方法的作用：
> - 基于父`context`创建一个新的`context`
> - 为一个指定的key设定值
> 
> 你可以简单理解为，`context`中包含了一个`map`，所以你可以根据key来对它进行添加或获取某个值。

`context.WithValue`功能比较强大，它可以携带任意类型的值，下面我们以一个例子来看一下如何通过`context`添加、获取数据。

```golang
package main

import (
	"context"
	"fmt"
)

func AddValue(ctx context.Context) context.Context {
	return context.WithValue(ctx, "keyGuan", "this is the value")
}

func GetValue(ctx context.Context, key string) interface{} {
	value := ctx.Value(key)

	return value
}

func main() {
	ctx := context.Background()
	ctx = AddValue(ctx)
	value := GetValue(ctx, "keyGuan")
	fmt.Println(value)
}
```
> `context`的设计哲学: 不变性
>
> 所有的context都会返回一个新的`context.Context`结构体。这就意味着你必须任何关于`context`操作的返回值，并且这些值可能会被一个新的`context`所覆盖。
> 
> 关于GoLang中不变性详情请见我后续的文章

使用这种技术，你可以将`context.Context`传递给其它的并发函数，只要你正确地管理所传递的上下文，这将是在这些并发函数之间共享作用域值的非常好的方法（这意味着每个上下文将在其作用域上保持自己的值）。这正是`net/http`包在处理HTTP请求时的做法。为了详细说明这一点，我们来看看下一个例子。

**中间件**

`request scoped data`的一个很好的例子是在Web请求处理程序中使用中间件。`http.Request`类型包含一个`context`，它可以在整个HTTP管道中携带`scoped data`。
在HTTP管道中添加中间件，然后将中间件的结果添加到`http.Request`的`context`中，这是非常常见的代码。

这是一个非常有用的技术，因为你可以在以后的阶段中，依靠那些在pipline中已经确切发生改变的东西。这也使你能够使用通用代码来处理http请求，同时也能满足你想要共享数据的范围（而不是共享全局变量上的数据）。下面是一个利用请求上下文的中间件的例子：

```golang
package main

import (
	"context"
	"fmt"
	"net/http"
	"github.com/google/uuid"
	"github.com/gorilla/mux"
)

func isAlive(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)

	id := r.Context().Value("id")
	fmt.Printf("[%v] Status: 200 - I am live!", id)
	w.Write([]byte("I am alive"))
}

func idMiddleware(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := uuid.New()
		r = r.WithContext(context.WithValue(r.Context(), "id", id))
		h.ServeHTTP(w, r)
	})
}
func main() {
	r := mux.NewRouter()
	r.Use(idMiddleware)
	r.HandleFunc("/isalive", isAlive).Methods(http.MethodGet)

	http.ListenAndServe(":8081", r)
}
```

### 2.2 带有cancellation的context

**说明**

`context`在GoLang中另外一个比较有用的功能是`cancellation`。当你需要发送一个**取消**信号时，这个非常有用。同时，当你收到一个**取消**信号时，你能将其传递下去，也是非常关键的。

例如，当你在一个函数中创建了上千个goroutine时，main函数将会一直等待所有的goroutine都执行完毕后或取消后才会继续向下执行；如果你收到了一个**取消**的信号，比较理想的做法是将之传递下去，这样你就不会浪费计算资源。

针对上面的这个例子，如果能够在不同的goroutine中共享同一个`context`的话，将会很容易实现上面的需求。

**使用**

可以使用`context.WithCancel(ctx)`创建一个带有cancellation功能的context。当需要**取消**的功能时，只需要调用cancel相关的函数就可以。

**示例**

假设有这样一个场景：我们向一个服务发送一个请求
- 如果超时后还没有返回response，我们将发送第二个请求
- 如果能收到任何一个response,所有的请求将会被取消

```golang
package main

import (
	"context"
	"fmt"
	"io/ioutil"
	"net/http"
	"time"
	"net/url"
)

func queryWithContext(urls []string) string{
	ch := make(chan string, len(urls))

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	for _, innerURL := range urls {
		go func(u string, c chan string) {
			c <- execWithContext(u, ctx)
		}(innerURL, ch)

		select {
		case r := <-ch:
			cancel()
			return r
		case <-time.After(100 * time.Second):

		}
	}

	return <-ch
}

func execWithContext(url string, ctx context.Context) string {
	start := time.Now()
	pURL, _ := url.Parse(url)
	req := &http.Request{URL: pURL}
	req = req.WithContext(ctx)

	if response, err := http.DefaultClient.Do(req); err == nil {
		defer response.Body.Close()
		body, _ := ioutil.ReadAll(response.Body)

		fmt.Printf("Requst: %d s from url %s\n", time.Since(start).Nanoseconds()/time.Second.Nanoseconds(), url)

		return fmt.Sprintf("%s from %s", body, url)
	} else {
		fmt.Println(err.Error())
		return err.Error()
	}
}
```

每一个请求都是在一个单独的goroutine中发起，所有的请求中都带有`context`，到此我们唯一需要做的就是把请求发送到客户端。当调用`cancel()`时，可以优雅的取消请求和底层的连接。

对于接受`context.Context`作为参数的函数来说，这是一个非常常见的模式。它们要么主动地对`context`进行操作（比如检查它是否被取消了），要么将它传递给处理它的底层函数（本例中是通过 `http.Request` 接收上下文的 `Do()` 函数）

### 2.3 带有timeout的context
这个使用起来比较简单：
```golang
ctx, cancel := context.WithTimeout(context.Background(), 500*time.Second)
```

### 2.4 gRPC
gRPC的实现也是依赖`context`的，通过它可以实现数据共享和流控，例如：取消工作流或请求。

> 更多关于gRPC相关的内容请查看官网。
