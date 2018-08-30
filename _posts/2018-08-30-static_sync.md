---
layout:     post
title:      将项目中的静态文件同步到静态服务器
subtitle:   借助Fabric实现的不同服务器文件同步
date:       2018-08-30
author:     yandenghong
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Python
    - Django
    - 部署
    - Fabric
---

## 前言
众所周知，django自带的服务器只是用作开发调试作用，在生产环境中，性能显然无法满足需求。因此，生产环境中，
大多都采用静态服务器+动态服务器+django的方式部署。项目中所有的静态文件交给静态服务器（比如nginx, 当然它不止这一个作用）,
动态请求就由动态服务器（比如uwsgi）来处理。

## 正文
有些线上的架构比较复杂，需要考虑到横向扩展性，因此会出现"一拖多"的架构，即最前面是一台安装了nginx的服务器，在其后可能有多台服务器，
每台上面部署着不同的服务，如果django项目部署在了这些处在后方的机器上， 那如果想沿用前言中提到的那种架构，就必须得同步本地的静态文件到
nginx所在的那台上去。

Fabric3 为我们封装了这个需求的实现。使用起来十分简单：
> 安装: pip install fabric3

example.py:
```python
    from fabric.contrib.project import rsync_project
    from fabric.api import env

    # 远程主机端口(可以不用设置, 默认是22,因为是基于ssh连接)
    env.port = '22'
    # 远程ip
    env.hosts = ['188.0.0.188']

    def func():
        # delete=True：删除本地已不存在的远程目录中的文件
        rsync_project(local_dir='你的本地文件目录', remote_dir='远程目录', delete=True)
```

### 执行同步
    fab -f example.py func

然后会提示你输入远程的密码（只在第一次执行时需要）
验证成功后，就开始同步了。

另外， 这个rsync_project是有检查机制的，每次执行只会同步发生了变化的文件，不会把整个目录copy过去。
这样只须每次在发布时执行一下上述命令。

现在，可以配置nginx和uwsgi去了。欲知后事如何，且看下回分解～～