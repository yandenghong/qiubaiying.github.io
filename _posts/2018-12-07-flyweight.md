---
layout:     post
title:      享元模式
subtitle:   结构型设计模式
date:       2018-11-26
author:     yandenghong
header-img: img/sky-min.jpg
catalog: true
tags:
    - 设计模式
    - python
---

# 场景

假设我们在开发一个大型的FPS游戏，这个游戏有一个开放宏大的由森林构成的3D世界，如果每棵树都是一个对象，那我们的系统将需要非常大的内存，因为，
如果直到耗尽服务器的物理内存还没有渲染完全部的树， 那么会将内存页替换到二级存储设备， 通常是硬盘。而硬盘和内存的性能的巨大的差异，这是不能接受的，
试想，如果渲染3D信息的速度不够快，那么，可能只有光秃秃的地面， 那给玩家带来的体验将是毁灭性的。

身为软件工程师，我们应该编写更好的软件来解决软件问题， 而不是要求客户或者公司购买更多更好的硬件。

因此，享元设计模式通过__为相似对象引入数据共享来最小化内存使用__，提升性能。


# 例子

构造一片水果树的森林， 无论构造的森林多大， 内存分配都保持相同。

# 实现

import enum
import random


class TreeType(enum.Enum):
    APPLE_TREE = "APPLE_TREE"
    CHERRY_TREE = "CHERRY_TREE"
    PEACH_TREE = "PEACH_TREE"
    PEAR_TREE = "PEAR_TREE"


class Tree:
    # 缓存池
    pool = dict()

    def __new__(cls, tree_type):
        obj = cls.pool.get(tree_type, None)
        # 缓存中没有才新建对象
        if not obj:
            obj = object.__new__(cls)
            # 新建后保存到缓存池中
            cls.pool[tree_type] = tree_type
        return obj

    def render(self, age, x, y):
        print("render a tree of type {} and age {} at ({}, {})".format(self.tree_type, age, x, y))


def main():
    rnd = random.Random()
    age_min, age_max = 1, 30
    min_point, max_point = 0, 100
    tree_counter = 0

    for _ in range(10):
        t1 = Tree(TreeType.CHERRY_TREE)
        t1.render(rnd.randint(age_min, age_max),
                  rnd.randint(min_point, max_point),
                  rnd.randint(min_point, max_point))
        tree_counter += 1

    for _ in range(3):
        t2 = Tree(TreeType.cherry_tree)
        t2.render(rnd.randint(age_min, age_max),
                  rnd.randint(min_point, max_point),
                  rnd.randint(min_point, max_point))
        tree_counter += 1

    for _ in range(5):
        t3 = Tree(TreeType.peach_tree)
        t3.render(rnd.randint(age_min, age_max),
                  rnd.randint(min_point, max_point),
                  rnd.randint(min_point, max_point))
        tree_counter += 1

    print('trees rendered: {}'.format(tree_counter))
    print('trees actually created: {}'.format(len(Tree.pool)))

    t4 = Tree(TreeType.cherry_tree)
    t5 = Tree(TreeType.cherry_tree)
    t6 = Tree(TreeType.apple_tree)
    print('{} == {}? {}'.format(id(t4), id(t5), id(t4) == id(t5)))
    print('{} == {}? {}'.format(id(t5), id(t6), id(t5) == id(t6)))


if __name__ == '__main__':
    main()

# 总结

在我们想要优化内存使用提高应用性能之时,可以使用享元。
在所有内存受限(想一想嵌入式系统)或关注性能的系统(比如图形软件和电子游戏)中,这一点相当重要。在应用需要创建大量的计算代价大但共享许多属性的对象时,
可以使用享元。重点在于将不可变(可共享)的属性与可变的属性区分开。
