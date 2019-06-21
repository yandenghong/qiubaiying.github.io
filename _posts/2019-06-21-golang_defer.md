---
layout:     post
title:      深入理解Golang defer执行顺序
date:       2019-06-21
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - Golang
---
## 前言

在Golang中，提供了defer机制来处理成对的操作,如打开、关闭、连接、断开连接、加锁、释放锁。通过defer机
制,不论函数逻辑多复杂,都能保证在任何执行路径下,资源被释放。

那么defer到底是如何运行的?仅仅是简单的函数执行完后调用吗?

## 正文
通过一个简单的统计函数执行时长的函数来看看其执行顺序:
```text
package main

import (
    "fmt"
    "time"
)


func trace(msg string) func() {
    start := time.Now()
    fmt.Printf("enter %s\n", msg)
    return func() {
        fmt.Printf("exit %s (%s)\n", msg,time.Since(start))
    }
}

func someOperation() {
    defer trace("someOperation")()
    fmt.Println("do some operation...")
    time.Sleep(3 * time.Second)
}

func main(){
    someOperation()
}

```
执行结果:
```text
enter someOperation
do some operation...
exit someOperation (3.000238625s)
```
从结果可以看出，在运行到defer时，先执行了defer函数内的代码，在遇到返回时，跳出，继续执行原来的代码，
待原来的函数执行完毕后，才执行defer函数的返回。

那么，如果defer函数没有返回值呢?

对上述代码稍作修改:

```text
package main

import (
    "fmt"
    "time"
)

func trace(msg string) {
    // 这里的trace 仅仅打印，没有返回
	fmt.Printf("enter %s\n", msg)
}

func someOperation() {
    // 对于没有返回值的函数，defer调用其时，后面不能加()调用符号
    defer trace("someOperation")
    fmt.Println("do some operation...")
    time.Sleep(3 * time.Second)
}

func main(){
    someOperation()
}

```

执行结果:
```text
do some operation...
enter someOperation
```
可以看到，当没有返回值时，defer函数内的代码会在原函数执行完毕后才执行。

## 总结
defer机制在其调用的函数没有返回值时，在原函数执行完毕后，才执行defer函数。
若调用的函数有返回，则会在执行原函数过程中遇到defer时，先执行defer返回值之前的代码，再执行原函数，原函数执行完毕后，
才返回defer函数的返回值，如果这个返回值是一个函数，即执行这个函数。