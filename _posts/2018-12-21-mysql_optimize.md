---
layout:     post
title:      Mysql优化之提高最大连接数
subtitle:   性能优化
date:       2018-12-21
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - 性能优化
    - mysql
    - 并发
---

## 前言

正常安装的mysql, 最大支持的连接数是151, 不适用于高并发场景。这个数远远没有到mysql的瓶颈, 因此我们需要优化。

本文以ubuntu16.04为例。

## 正文

### 修改mysql配置

```
sudo vim /etc/mysql/mysql.conf.d/mysqld.conf
```

找到**max_connections**项并去掉注释，修改该值为需要的值。

:x保存



此时重启mysql服务后你会发现最大连接数确实变了，但是没有变成你刚才设置的值，而是214：

![](/img/mysql_2.png)

> 图为：仅修改mysql配置后的最大连接数

还需要继续优化

### 声明

搜索引擎里确实有很多这方面的资料，但是有些资料的质量实在不敢恭维。

有些说ubuntu16.04还需要去在/etc/security/limits.conf 文件里加两行:

```
* hard nofile 65535

* soft nofile 65535
```

这完全是多此一举，ubuntu从版本15.04开始，就不再遵守/etc/security/limits.conf中对系统服务的限制了。 因此，不用执行这一步。


###  修改mysql服务限制
mysql服务限制在Systemd配置文件中定义，要令其生效，先复制到/etc/systemd中，然后编辑。

```
sudo cp /lib/systemd/system/mysql.service /etc/systemd/system/

sudo vim /etc/systemd/system/mysql.service
```

在文件底部添加这两行:
```
LimitNOFILE=infinity  # 这个值也可以设置为你刚才在mysql配置里设置的最大连接数
LimitMEMLOCK=infinity
```

:x保存

### 重新加载systemd配置
```
sudo systemctl daemon-reload
```

### 重启mysql
```
sudo /etc/init.d/mysql restart
```

重启成功后，你应该可以看到:
![](/img/mysql_restart.png)

> 图为：mysql重启成功


### 优化成功

再去查看最大连接数, 我之前设置了8192:
![](/img/mysql_3.png)

> 图为：优化后的mysql最大连接数