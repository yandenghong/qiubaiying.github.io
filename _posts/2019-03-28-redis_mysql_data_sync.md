---
layout:     post
title:      最小成本实现redis和mysql数据双向同步
subtitle:   基于请求触发的状态检查
date:       2019-03-28
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - Django
    - redis
    - mysql
    - 数据库
---

## 前言

最近遇到了一个项目中某些表的部分数据被高频访问的问题。原本的数据库是mysql,为了优化性能, 所以选择了高性能的
redis作为这部分高频访问的数据的缓存数据库。另外，项目对缓存和数据库的数据同步有要求。因此，两者需要双向同步。

## 正文
缓存的数据流通常都是数据库发生更新（新增，修改，删除）时，更新缓存的数据。查询时先查缓存，缓存中没有，
再去查数据库，查到了，再添加到缓存中。

而我的项目中并不完全符合这种逻辑。首先我的数据有过期时间，这个过期时间是依赖于redis来做的，也就是说我在往缓存写
数据时添加了过期时间，那么问题自然而然的来了，当缓存中的数据过期了，我本地的数据该怎么与其保持一致？
这就涉及到缓存向数据库同步了。

### 数据库向缓存同步
首先来看看数据库向缓存同步我是如何做的:

Django内置了一个模块[signals][1], 该模块的作用是接收数据库传递的信号，根据信号执行一些操作。
Django已经封装了数据库增删改查的信号。那么数据库向缓存同步，假设我的表只需一个新增操作，那我只需要接收一个对应表的
新增信号，并在接收后，把需要缓存的数据，存到缓存即可(python操作redis的库`pip install redis`)。

例如:
```python
from django.db import models
from django.db.models.signals import post_save
from django.dispatch import receiver

from my_utils import redis_cli


class Book(models.Model):
    name = models.CharField("名称", max_length=16, db_index=True)
    category = models.CharField("种类", max_length=16, db_index=True)
    
# post_save 即写入/新增的信号
@receiver(post_save, sender=Book)
def create_map_cache(sender, instance=None, created=False, **kwargs):
    """
    当Book`写入`时，将该条数据的name, category存入redis中
    """
    if created:
        #　将该条数据的name, category存入redis中, 过期时间1个小时
        redis_cli.set_data(instance.name, instance.category, 60*60)
```

完美实现数据库向缓存同步。

### 缓存向数据库同步
因为我在缓存中的数据是有过期时间的，我需要这些数据失效了之后，我能在数据库层面也将其标记为过期。
这里也有很多解决方案，但搜索出来的方案都是成本比较高的，所以我没有采用。这里结合我的具体业务场景，
我想出了一个轻量的解决方案，这个方案最大的特点就是*惰性*。

我在原表中新增了两个字段: `is_cached`, `is_expired`。 `is_cached` 用于标记该数据是否已被缓存过。
`is_expired`用于标记是否过期。这两个字段的默认值都是`False`。

有了这两个字段，我在向缓存写入时，写入完毕将当前数据的`is_cached`置为`True`。
那下次如果一个请求过来，请求已经在缓存中过期的数据，缓存中没有，而数据库中有，并且根据`is_cached`是`True`，
我即可确定这条记录是过期了，然后此时再将该条数据的`is_expired`置为`True`。

之所以说这是一个*惰性*的方案，就是因为是基于请求触发的状态检查。

像这样:
```python
from django.db import models
from django.db.models.signals import post_save
from django.dispatch import receiver

from my_utils import redis_cli


class Book(models.Model):
    name = models.CharField("名称", max_length=16, db_index=True)
    category = models.CharField("种类", max_length=16, db_index=True)
    # 新增的两个字段
    is_cached = models.BooleanField("是否已被缓存", default=False)
    is_expired = models.BooleanField("是否已过期", default=False)


# post_save 即写入/新增的信号
@receiver(post_save, sender=Book)
def create_map_cache(sender, instance=None, created=False, **kwargs):
    """
    当Book`写入`时，将该条数据的name, category存入redis中, 并将is_cached置为True.
    """
    if created:
        #　将该条数据的name, category存入redis中, 过期时间1个小时
        redis_cli.set_data(instance.name, instance.category, 60*60)
        # instance就是写入的数据，这里直接更新其is_cached字段.
        instance.is_cached = True
        instance.save()
```

## 总结
个人结合特定的业务场景想出的解决方案，如果能帮到你，我很开心。

[1]: https://docs.djangoproject.com/en/2.1/topics/signals/
