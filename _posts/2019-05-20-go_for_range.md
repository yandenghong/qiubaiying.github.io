---
layout:     post
title:      Golang中for range的坑
date:       2019-05-20
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - Golang
---
## 前言
最近遇到一个golang for range时的坑，记录一下。

## 正文
遍历一个切片，并将切片的值当成映射的键和值存入，切片类型是一个int型，映射的类型是键为int型，值为*int，即值是一个地址。

原来我是这样写的:
```text
package main

import "fmt"

func main() {
    slice := []int{0, 1, 2, 3}
    myMap := make(map[int]*int)

    for index, value := range slice {
        myMap[index] = &value
    }
    fmt.Println("=====new map=====")
    prtMap(myMap)
}

func prtMap(myMap map[int]*int) {
    for key, value := range myMap {
        fmt.Printf("map[%v]=%v\n", key, *value)
    }
}
```

预期输出:
```text
=====new map=====
map[0]=0
map[1]=1
map[2]=2
map[3]=3
```

实际输出:
```text
=====new map=====
map[0]=3
map[1]=3
map[2]=3
map[3]=3
```

看实际输出发现，映射的元素的值都是3.这其中发生了什么?

遍历到切片的最后一个元素3时，将3写入了该地址，所以导致映射所有值都相同。

### 对切片for range 的底层代码
```text
for_temp := range
len_temp := len(for_temp)
for index_temp = 0; index_temp < len_temp; index_temp++ {
       value_temp = for_temp[index_temp]
       index = index_temp
       value = value_temp
       original body
}
```

通过源码再去看上面的代码，遍历后的值赋给了`value`，而在上面代码中，会把`value`的地址保存到`myMap`的值中。这里的`value`是个全局变量，所以赋完值之后`myMap`里面所有的值都是`value`，所以结构都是一样的而且是最后一个值。

注意，这里必须是保存指针才会有问题，如果直接保存的是`value`，因为 Golang 是值拷贝，所以值会重新复制再保存，这种情况下结果就会是正确的了。
