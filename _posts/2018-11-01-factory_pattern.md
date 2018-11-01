---
layout:     post
title:      工厂模式
subtitle:   创建型设计模式
date:       2018-11-01
author:     yandenghong
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 设计模式
    - python
---

## 什么是工厂模式
在工厂设计模式中，客户端可以请求一个对象，而无需知道使用哪个类来创建这个对象。工厂背后的思想是**简化对象的创建**。与客户端自己基于类实例化直接
创建对象相比，基于一个**中心化**函数来实现，更易于追踪创建了哪些对象。通过将创建对象的代码和使用对象的代码解耦，工厂能够降低应用维护的复杂度。

工厂通常有两种形式：一种是**工厂方法，对不同的输入参数返回不同的对象**;第二种是**抽象工厂，它是一组用于创建一系列相关事物对象的工厂方法**。

### 工厂方法
    在工厂方法模式中，执行单个函数，传入一个参数（用于表明我们想要什么），但并不要求知道任何关于对象如何实现以及对象来自哪里的细节。最熟悉的例子
就是Django的forms模块：支持不同种类字段的创建和定制。

#### 何时使用
场景一： 在项目中创建对象的代码分布在多个不同的地方，而不是仅在一个函数中，没法跟踪这些对象。

场景二： 将对象的创建和使用解耦。

#### 实现
有一些输入数据存储在一个XML文件和一个JSON文件中,要对这两个文件进行解析,获取一些信息。同时,希望能够对这些(以及将来涉及的所有)外部服务进
行集中式的客户端连接。

```python
import xml.etree.ElementTree as etree
import json


class JsonConnector:
    def __init__(self, filepath):
        self.data = dict()
        with open(filepath, mode='r', encoding='utf-8') as f:
            self.data = json.load(f)

    @property
    def parsed_data(self):
        return self.data


class XMLConnector:
    def __init__(self, filepath):
        self.tree = etree.parse(filepath)

    @property
    def parsed_data(self):
        return self.tree


class Connection:
    def _connection_factory(self, filepath):
        """工厂方法
            基于输入文件路径的扩展名返回一个JsonConnector或XMLConnector的实例
        """
        if filepath.endswith('json'):
            connector = JsonConnector
        elif filepath.endswith('xml'):
            connector = XMLConnector
        else:
            raise ValueError('Cannot connect to {}'.format(filepath))
        return connector(filepath)

    def connect_to(self, filepath):
        """对上面的工厂方法进行了包装，增加了异常处理
        """
        factory = None
        try:
            factory = self._connection_factory(filepath)
        except ValueError as ve:
            print(ve)
        return factory

```

### 抽象工厂
一个抽象工厂是逻辑上的一组工厂方法，其中每个工厂方法负责产生不同种类的对象。
其好处和工厂方法是相同的：让对象的创建更容易追踪;将对象创建与使用解耦;提供优化内存占用和应用性能的潜力。

#### 实现
**此处借鉴了Python 3 Patterns & Idioms(Bruce Eckel著)一书中的一个例子。我对其部分代码做了改进。**

想象一下,我们正在创造一个游戏,或者想在应用中包含一个迷你游戏让用户娱乐娱乐。我们希望至少包含两个游戏,一个面向孩子,一个面向成人。
在运行时,基于用户输入,决定该创建哪个游戏并运行。游戏的创建部分由一个抽象工厂维护。

```python
class Base:
    role = None


class BaseCharacter(Base):
    role = "character"


class BaseObstacle(Base):
    role = "obstacle"


class Frog(BaseCharacter):
    """青蛙:主人公一"""
    def __init__(self, name):
        self.name = name

    def __str__(self):
        return self.name

    def interact_with(self, obstacle):
        """交互方法"""
        print("青蛙{}遭遇了{}并且{}".format(self.name, obstacle, obstacle.action()))


class Wizard(BaseCharacter):
    """巫师:主人公二"""
    def __init__(self, name):
        self.name = name

    def __str__(self):
        return self.name

    def interact_with(self, obstacle):
        """交互方法"""
        print("巫师{}遭遇了{}并且{}".format(self.name, obstacle, obstacle.action()))


class Bug(BaseObstacle):
    """虫子:障碍一"""
    def __str__(self):
        return "一个虫子"

    @staticmethod
    def action():
        return "吃掉了它"


class Orc(BaseObstacle):
    """兽人:障碍二"""
    def __str__(self):
        return "一个邪恶的兽人"

    @staticmethod
    def action():
        return "禁锢了它"


class World:
    """抽象工厂: 创建游戏的主人公和障碍物"""
    def __init__(self, player_class, name: str):
        print(self)
        self._check_is_character(player_class)
        self.player_class = player_class
        self.player_name = name

    @staticmethod
    def _check(player_class):
        if not hasattr(player_class, 'role'):
            raise ValueError("目标类错误")

    def _check_is_character(self, player_class):
        self._check(player_class)
        if player_class.role != "character":
            raise ValueError("非主人公类")
        return player_class.role

    def __str__(self):
        return "\n\n\t-------- World--------"

    def make_character(self):
        """创建游戏主人公"""
        return self.player_class(self.player_name)

    def make_obstacle(self):
        """创建障碍物"""
        if self.player_class.__name__ == "Wizard":
            return Orc()
        elif self.player_class.__name__ == "Frog":
            return Bug()
        else:
            raise ValueError("无效的主人公类")


class GameLauncher:
    """游戏启动器"""

    def run(self):
        name = self._get_name()
        result = self._check_age(name)
        if not result:
            player_class = Frog
        else:
            player_class = Wizard
        self._run(player_class, name)

    @staticmethod
    def _run(player_class, name):
        world = World(player_class=player_class, name=name)
        hero = world.make_character()
        obstacle = world.make_obstacle()
        hero.interact_with(obstacle)

    @staticmethod
    def _get_name():
        while True:
            name = input("请为自己的角色起个名称")
            if name:
                break
        return name

    @staticmethod
    def _check_age(name):
        while True:
            age = input("欢迎你!{}!请告诉我你的年龄!".format(name))
            try:
                age = int(age)
            except ValueError:
                print("请输入有效的年龄")
            if isinstance(age, int):
                break
        if age < 18:
            return False
        else:
            return True


launcher = GameLauncher()


if __name__ == '__main__':
    launcher.run()
```

## 总结
工厂设计模式的工厂方法模式和抽象工厂设计模式，都可以用于以下几种场景:

* 1.想要追踪对象的创建时

* 2.想要将对象的创建和使用解耦时

* 3.想要优化应用的性能和资源占用时

工厂设计模式的核心就是**一个不属于任何类的单一函数， 负责单一种类对象的创建**。
