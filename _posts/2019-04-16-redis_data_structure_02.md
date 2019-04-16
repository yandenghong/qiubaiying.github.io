---
layout:     post
title:      Redis数据结构之哈希(Hash)类型
subtitle:   Redis数据结构
date:       2019-04-16
author:     yandenghong
header-img: img/post-bg-star-trails.jpg
catalog: true
tags:
    - 数据结构
    - redis
---

redis的**Hash**是一个string类型的field和value的映射表。极其适合存储对象。

> 每个hash可以存储40多亿键值对。


## Hash数据结构
```c
/*Hash表一个节点包含Key,Value数据对 */
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; /* 指向下一个节点, 链接表的方式解决Hash冲突 */
} dictEntry;

/* 存储不同数据类型对应不同操作的回调函数 */
typedef struct dictType {
    unsigned int (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

typedef struct dictht {
    dictEntry **table; /* dictEntry*数组,Hash表 */
    unsigned long size; /* Hash表总大小 */
    unsigned long sizemask; /* 计算在table中索引的掩码, 值是size-1 */
    unsigned long used; /* Hash表已使用的大小 */
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2]; /* 两个hash表,rehash时使用*/
    long rehashidx; /* rehash的索引, -1表示没有进行rehash */
    int iterators; /*  */
} dict;
```

## Hash结构图
![](/img/hash.png)


## 常用操作命令

创建哈希:
```text
HMSET key field1 value1 [field2 value2 ... fieldn valuen]
```

获取指定字段的值:
```text
HGET key field
```

获取所有字段和值:
```text
HGETALL key
```

查看指定字段是否存在:
```text
HEXISTS key field
```

删除一个或多个字段:
```text
HDEL key field1 [field2, ... fieldn]
```

给哈希的一个字段赋值:
```text
HSET key field value
```

获取所有的值:
```text
HVALS key
```
