---
layout:     post
title:      修饰器模式
subtitle:   结构型设计模式
date:       2018-11-15
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - 设计模式
    - Python
---

## 介绍
无论何时我们想对一个对象添加额外的功能，不外乎有三种方法：

* 直接将功能添加到对象所属的类(添加一个新的方法)
* 使用组合
* 使用继承

第一种直接添加的方法，不符合开放/封闭原则，因此很少采用。而第三种，继承，会使得代码更难复用，继承关系是静态的，并且应用于整个类以及这个类的所有实例。
所以，通常优先选择组合。

修饰器模式，就是一种组合，支持动态扩展一个对象的功能，它能在不影响其他对象的情况下， 动态地将功能添加到一个对象中。在python里，内置了修饰器特性。
python 修饰器是一个可调用对象(函数，方法，类), 接受一个函数对象作为输入，并返回另一个函数对象。任何有这个属性的可调用的对象都可以作为一个修饰器。
典型的例子：内置的property修饰器，可以把一个方法表现为一个变量。

需要注意的一点是，修饰器模式和python修饰器之间并不是一对一的等价关系。__python修饰器能做的实际上比修饰器模式多得多。__

## 实现

### 递归算法实现斐波那契数列(原始版本)
```python

def fibonacci(n):
    """递归实现斐波那契数列"""
    assert (n >= 0), "n must be >= 0"
    return n if n in (0, 1) else fibonacci(n-1) + fibonacci(n-2)


if __name__ == '__main__':
    from timeit import Timer
    t = Timer("fibonacci(8)", "from __main__ import fibonacci")
    print(t.timeit())

```

耗时:
![](/img/time_ret1.png)
> 图为：计算第8个斐波那契数列耗时

###  递归算法实现斐波那契数列(缓存版)
```python

# 缓存已经计算好的数值
known = {0: 0, 1: 1}


def fibonacci(n):
    """递归实现斐波那契数列"""
    assert (n >= 0), "n must be >= 0"
    if n in known:
        return known[n]
    res = fibonacci(n-1) + fibonacci(n-2)
    known[n] = res
    return res


if __name__ == '__main__':
    from timeit import Timer
    # 由计算第8个改为计算第100个
    t = Timer("fibonacci(100)", "from __main__ import fibonacci")
    print(t.timeit())

```

耗时:
![](/img/time_ret2.png)
> 图为：缓存改造后，计算第100个斐波那契数列的耗时

### 比较
可以看到，在用缓存改造后，性能得到了巨大提升。

但这个方法有一些问题，性能确实提高了，但代码却没有原始版本那么简洁了。

### 递归算法实现斐波那契数列(缓存升级版)
```python
import functools


def memoize(func):
    """缓存修饰器"""
    known = dict()

    # 使用内置的wraps 保留被修饰函数的文档和名称
    @functools.wraps(func)
    def inner_memoize(*args):
        if args not in known:
            known[args] = func(*args)
        return known[args]
    return inner_memoize


@memoize
def fibonacci(n):
    """递归实现斐波那契数列"""
    assert (n >= 0), "n must be >= 0"
    return n if n in (0, 1) else fibonacci(n-1) + fibonacci(n-2)

```

使用了修饰器模式后， 不仅增强了原版的性能，而且保持了代码的简洁，还有一个巨大的好处就是此类递归计算的函数，都可以用该修饰器来扩展。


## 总结

修饰器模式允许我们无需使用继承或组合就能扩展任意可调用对象(函数、方法或类)的行为。遵循开放/封闭原则，扩展功能的同时保证原有代码不变。