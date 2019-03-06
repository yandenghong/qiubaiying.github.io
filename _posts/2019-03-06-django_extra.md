---
layout:     post
title:      Django 高级技巧之Queryset修改机制-extra()
subtitle:   
date:       2019-03-06
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - Django
    - SQL
    - 数据库
---

## 前言
Django ORM 为我们提供了极其方便的数据访问方式。 在大多数时候，对于查询集，
我们用到的都是诸如filter(), order_by(), get()等等的方法即可满足项目需求。

当遇到下面这个需求：

有一张表A， 其中有个字段是个有关规模的枚举字段scale，可选值有小，较小，中等，较大，大。
现在有一个接口需要将该表中的数据按照这个scale降序排序.

该如何做呢？在SQL语句中在 ORDER BY 子句中使用 FIELD 函数可以实现这一点. 它的工作方式是指定要排序的列, 然后按顺序排序它们的值.
但是用Django ORM呢?

## 正文

Django 早在1.0版本就加入了Queryset的extra()方法，该方法能在QuerySet 生成的 SQL 从句中注入新子句，从而实现Queryset修改。

```python
def extra(self, select=None, where=None, params=None, tables=None,
          order_by=None, select_params=None):
    """Add extra SQL fragments to the query."""
    assert self.query.can_filter(), \
        "Cannot change a query once a slice has been taken"
    clone = self._chain()
    clone.query.add_extra(select, select_params, where, params, tables, order_by)
    return clone
```
#### select

select 参数可以让你在 SELECT 从句中添加其他字段信息。它应该是一个字典，存放着属性名到 SQL 从句的映射。

例如：

Entry.objects.extra(select={'is_recent': "pub_date > '2006-01-01'"})
结果中每个 Entry 对象都有一个额外的 is_recent 属性，它是一个布尔值，表示 pub_date 是否晚于2006年1月1号。

Django 会直接在 SELECT 中加入对应的 SQL 片断，所以转换后的 SQL 如下：

SELECT blog_entry.*, (pub_date > '2006-01-01')
FROM blog_entry;

#### where/tables

where 参数显示定义 SQL 中的 WHERE 从句。tables 手动给 SQL FROM 从句添加其他表。

where 和 tables 都接受字符串列表做为参数。所有的 where 参数彼此之间都是 "AND" 关系。

例如：

Entry.objects.extra(where=['id IN (3, 4, 5, 20)'])
...大致可以翻译为如下的 SQL:

SELECT * FROM blog_entry WHERE id IN (3, 4, 5, 20);
下面这个是 tables 的例子:

queryset.extra(tables=['(select * from table) as k'])
翻译的 SQL 如下：

select ........ from `self_table`, `(select * from table) as k`

这展示了如何运行子查询。

在使用 tables 时，如果你指定的表在查询中已出现过，那么要格外小心。当你通过 tables 参数添加其他数据表时，如果这个表已经被包含在查询中，那么 Django 就会认为你想再一次包含这个表。这就导致了一个问题：由于重复出现多次的表会被赋予一个别名，所以除了第一次之外，每个重复的表名都会分别由 Django 分配一个别名。所以，如果你同时使用了 where 参数，在其中用到了某个重复表，却不知它的别名，那么就会导致错误。

#### order_by

如果你已通过 extra() 添加了新字段或是数据库，此时若想对新字段进行排序，就可以给 extra() 中的 order_by 参数传递一个排序字符串序列。字符串可以是 model 原生的字段名(与使用普通的 order_by() 方法一样)，也可以是 table_name.column_name 这种形式，或者是你在 extra() 的 select 中所定义的字段。

例如：

q = Entry.objects.extra(select={'is_recent': "pub_date > '2006-01-01'"})

q = q.extra(order_by = ['-is_recent'])

这段代码按照 is_recent 对记录进行排序，字段值是 True 的排在前面，False 的排在后面。(True 在降序排序时是排在 False 的前面)。

顺便说一下，上面这段代码同时也展示出，可以依你所愿的那样多次调用 extra() 操作(每次添加新的语句结构即可)。

#### params
Bad:
```python
Entry.objects.extra(where=["headline='Lennon'"])
```

Good:
```python
Entry.objects.extra(where=['headline=%s'], params=['Lennon'])
```

现在回到上面的需求,我们只需这样写:
```python
ordering = 'FIELD(`scale`, {})'.format(','.join(["'{}'".format(i) for i in ["低", "较低", "中", "较高", "高"]]))
data = DemoDatabaseClass.objects.extra(select={'ordering': ordering}, order_by=('-ordering',))
```

## 总结
优点: 简练地表达复杂的 WHERE 子句。

缺点：自定义的查询不能在不同的数据库之间兼容(因为手写 SQL代码的原因)，而且违背了 DRY 原则，所以如非必要，还是尽量避免写 extra。