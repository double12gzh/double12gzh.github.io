---
layout: post 
title: 在GoLang中实现线程安全的字典
categories: [golang]
description: golang,map,thread-safe
keywords: golang,map,thread-safe
excerpt: 摘要：本文主要解释如何通过RWMutex来实现一个基于内存的`字典`数据结构。
---


**目录**

* TOC
{:toc}


![](https://gitee.com/double12gzh/wiki-pictures/raw/master/Go-How-to-Reduce-Lock-Contention-with-the-Atomic-Package/%E5%9B%BE0.png)

## 1. 背景
本文主要解释如何通过RWMutex来实现一个基于内存的`字典`数据结构。

在项目中，经常需要与并发打交道，这其中很难避免会遇到多个并发的用户同时获取内存中数据的情况，因此我们必须能够有一种方式，可以对这些数据进行读写并发控制。

## 2. 实现
## 2.1 数据结构定义
为了达到我们的需求，我设计了以下的自定义的数据结构
```golang
package dictionary

import "sync"

type iKey interface{}
type iValue interface{}

type Dictionary struct {
    items map[iKey]iValue
    lock sync.RWMutex
}
```

> 对于上面的结构作如下说明：
> - `items`用于保存所有的key-value数据
> - `lock`用于控制用户对`items`的读写操作

> 对于RWMutex作如下说明：
> `Lock()`：每次只允许有一个goroutine在同一时刻获取`读写`锁
> `RLock()`: 同一时刻可以有多个goroutine获取`读`锁

## 2.2 方法实现
接下来，我们实现一下这个`Dictionary`中所支持一些操作。

### 2.2.1 添加key-value
```golang
// Add 向dictionary中添加一个新的key/value
func (d *Dictionary) Add(key iKey, value iValue) {
	d.lock.Lock()
	defer d.lock.Unlock()
	
	if d.items == nil {
		d.items = make(map[iKey]iValue)
	}
	
	d.items[key] = value
}
```
> 说明: 
> 
> - 在向dictionary中添加新的key/value前，先通过`d.lock.Lock()`获取到一个锁，这样可以有效避免一些误操作。当插入成功后，通过`d.lock.Unlock()`把刚才获取到锁释放掉。
> - 如果一个请求已经获取到了`Lock()`锁，那么另外一个想要获取`RLock()`的请求将不得不等待第一个请求释放锁(`Unlock()`)

### 2.2.2 删除key-value
```golang
func (d *Dictionary) Remove(key iKey) bool{
	d.lock.Lock()
	defer d.lock.Unlock()
	
	if _, ok := d.items[key]; ok {
		delete(d.items, key)
	}
	
	return true
}
```
> 思路同`Add`这里不再多讲。
> 删除操作可以看成是一个`写`操作，因此，同样需要使用`Lock()`和`Unlock()`。

### 2.2.3 获取key-value
```golang
func (d *Dictionary) Get(key iKey) iValue {
	d.lock.RLock()
	defer d.lock.RUnlock()
	
	return d.items[key]
}
```

> 需要强调一点是，如果仅仅是在多个goroutine中并发的去`读取`数据，那么将不会存在数据竞争的问题。如果我们需要获取`Lock()`锁，那么必须等到执行完`RUnlock()`才可以。

### 2.2.4 key是否存在
```golang
func (d *Dictionary) Exist(key iKey) bool {
	d.lock.RLock()
	defer d.lock.RUnlock()
	
	if _, ok := d.items[key]; ok {
		return true
	} else {
		return false
	}
}
```
> 这个操作我们认为他是一个`读`操作。因此这里使用的是`RLock()`和`RUnlock()`

### 2.2.5 清空所有key-value
```golang
func (d *Dictionary) Clear() {
	d.lock.Lock()
	defer d.lock.Unlock()
	d.items = make(map[iKey]iValue)
}
```
> 这个操作需要获取`Lock()`和`Unlock()`

### 2.2.6 获取字典中元素的个数
```golang
func (d *Dictionary) Size() int {
	d.lock.RLock()
	defer d.lock.RUnlock()
	
	return len(d.items)
}
```

> 需要先获得读锁

### 2.2.7 获取字典中所有的key
```golang
func (d *Dictionary) GetKeys() []iKey {
	d.lock.RLock()
	defer d.lock.RUnlock()
	
	tempKeys := make([]iKey, 0)
	for i := range d.items {
		tempKeys = append(tempKeys, i)
	}
	
	return tempKeys
}
```

### 2.2.8 获取字典中所有的value
```golang
func (d *Dictionary) GetValues() []iValue {
	d.lock.RLock()
	defer d.lock.RUnlock()
	
	tempValues := make([]iValue, 0)
	for _, v := range d.items {
		tempValues = append(tempValues, v)
	}
	
	return tempValues
}
```

## 3. 小结
GoLang通过内置变量`map`和包`sync.RWMutex`提供了一个非常方便的实现字典的方法。`map`并不是线程安全的。
