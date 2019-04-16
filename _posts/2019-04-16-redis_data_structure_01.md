---
layout:     post
title:      Redis数据结构之字符串(String)类型
subtitle:   Redis数据结构
date:       2019-04-16
author:     yandenghong
header-img: img/post-bg-ocean.jpg
catalog: true
tags:
    - 数据结构
    - redis
---

**String**是redis中最基本的一种数据类型。其结构为key-value键值对。

常用的操作命令:

赋值:
```text
SET key value
```
> key: 键名 ,  value:对应的值。


取值:
```text
GET key
```
> 获取key对应的值，如果没有该key,则返回nil。

删除:
```text
DEL key
```
> 该命令可用于所有类型。

赋值同时设置过期时间:
```text
SETEX key seconds value
```
> 设置一个键为key值为value的键值对，seconds秒后过期。

数值加1:
```text
INCR key
```
> 如果key对应的值为空，则默认从0加1，如果值无法转为数字，则报异常。

数值减1:
```text
DECR key
```
> 如果key对应的值为空，则默认从0减1，如果值无法转为数字，则报异常。

数值加指定值:
```text
INCRBY key num
```
> 将key对应的值加上num, 值若为空则默认从0加。

数值减去指定值:
```text
DECRBY key num
```
> 将key对应的值减去num, 值若为空则默认从0减。
