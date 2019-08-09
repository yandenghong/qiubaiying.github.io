---
layout:     post
title:      删除嵌套json中的空字段
date:       2019-08-09
author:     yandenghong
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - 算法
    - Python
---

### 样本
```json
{
        "name": "gene",
        "age": 18,
        "times": [1,2,3,4,5],
        "books": {
            "a": null,
            "b":"yitiantulongji"
        },
        "nums": [],
        "friends": {
            "f": null,
            "g": {
                "my_num":2
            }
        }
    }
```

### 需求
删除掉上述数据中所有空字段。

### 实现
这里为了代码清晰程度， 省去了json转为dict的代码。

```python
def clean_empty(d):
    if not isinstance(d, (dict, list)):
        return d
    if isinstance(d, list):
        return [v for v in (clean_empty(v) for v in d) if v]
    return {k: v for k, v in ((k, clean_empty(v)) for k, v in d.items()) if v}
```
处理嵌套问题的通用解法就是**递归**。这里首先针对有些字段是list或者dict类型，做了类型判断和
相应的处理，然后对嵌套结构的每一个值进行递归调用，结束条件即传入的值不再是dict或者list.

### 执行结果

```json
{
  "times": [1, 2, 3, 4, 5], 
  "age": 18, 
  "books": {
              "b": "yitiantulongji"
           }, 
  "name": "gene", 
  "friends": {
                "g": {
                        "my_num": 2
                      }
             }
}

```
