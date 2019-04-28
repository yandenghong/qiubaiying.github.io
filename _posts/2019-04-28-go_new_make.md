---
layout:     post
title:      Golang中new和make的区别
date:       2019-04-28
author:     yandenghong
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Golang
---

## 前言
在go语言中，分配内存的内建方法有两个: `new(T)`和`make(T, args)`。

本篇将详细说明二者的区别。

## 正文
`make(T, args)`只用于创建切片(slice)，映射(map)和信道(channel), 并返回类型为`T`的一个已初始化的值。

之所以这样做，是因为以上三种类型均属于引用数据类型，在使用之前，必须要初始化。

以切片(slice)为例:

切片是一个具有三项内容的描述符,包含一个指向(数组内部)数据的指针(*ptr)、长度(len)以及容量(cap)。

![](/img/go_slice01.png)

> 图为：go中的切片的结构

在这三项被初始化之前,该切片为`nil`。对于切片、映射和信道,make用于初始化其内部的数据结构并准备好将要使用的值。例如:
```go
make([]int, 10, 100)
```

上面的代码会分配一个100个int的数组空间,接着创建一个长度为10,容量为100并指向该数组中前10个元素的切片结构。而:
```go
new([]int)
```

会返回一个指向新分配的,已置零的切片结构,即一个指向`nil`切片值的指针。


下面的例子阐述`new`和`make`的区别:

```text
　

var	p	*[]int	=	new([]int)							//	allocates	slice	structure;	*p	==	nil;	rarely	useful

var	v	[]int	=	make([]int,	100)	//	the	slice	v	now	refers	to	a	new	array	of	100	ints

//	Unnecessarily	complex:
var	p	*[]int	=	new([]int)

*p	=	make([]int,	100,	100)

//	Idiomatic:
v	:=	make([]int,	100)

```

## 总结

`make`只适用于切片(slice)，映射(map)和信道(channel)且不返回指针。若要获得明确的指针,请使用`new`分配内存。

