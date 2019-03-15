---
layout:     post
title:      nginx内置的4种负载均衡算法
subtitle:   
date:       2019-03-15
author:     yandenghong
header-img: img/post-bg-u2.png
catalog: true
tags:
    - nginx
---

## 前言
首先说下nginx配置参数中的`upstream`，该参数用于负载均衡的配置，指定一组被代理的服务器，指定其分配方式。
```text
upstream demo_server { 
      server 192.168.1.121:2333;
      server 192.168.1.122:2333;
    }
    
server {
    ....
    location  / {         
       proxy_pass  http://demo_server;  # 请求转向demo_server定义的服务器列表         
    } 

```
## 正文
### 备份
假如有2台服务器，当一台服务器挂了，才启用第二台服务器给提供服务。第二台相当于一个备用服务器。
```text
upstream demo_server { 
      server 192.168.1.120:2333; 
      server 192.168.1.121:3333 backup;  # 备用服务器     
    }
```
### 轮询
nginx的默认负载均衡方式就是轮询，所有的服务器的权重都为1。所有服务器接受的请求数量是相同的。
```text
upstream demo_server { 
      server 192.168.1.120:2333; 
      server 192.168.1.121:3333;    
    }
```

### 加权轮询
根据每个服务器的权重来决定分发给每个服务器多少数量的请求, 参数为`weight`。
如果按照如下这样设置权重了，则权重为1的服务器处理三分之一的请求，权重为2的处理三分之二的请求。
```text
upstream demo_server { 
      server 192.168.1.120:2333 weight=1; 
      server 192.168.1.121:3333 weight=2;    
    }
```
### ip_hash
每个请求按访问ip的hash结果分配，这样每个访客固定访问一个服务器，可以解决session不同步的问题。
```text
upstream demo_server { 
      server 192.168.1.120:2333; 
      server 192.168.1.121:3333;
      ip_hash;    
    }
```
### 可选的状态参数
* `down`: 表示当前的服务器暂时不参与负载均衡。
* `backup`: 表示当前服务器为备用服务器。
* `max_fails`: 允许请求失败的次数，默认为1。当超过最大次数时，返回`proxy_next_upstream` 模块定义的错误。
* `fail_timeout`: 当在`fail_timeout`的时间内，某个server连接失败了`max_fails`次，则nginx会认为该server不工作了。同时，在接下来的`fail_timeout`时间内，nginx不再将请求分发给失效的server。

```text
upstream demo_server { 
      server 192.168.1.120:2333 weight=1 max_fails=2 fail_timeout=2; 
      server 192.168.1.121:3333 weight=2 max_fails=2 fail_timeout=2;
      ip_hash;    
    }
```
## 结语
以上就是nginx内置的4种负载均衡算法的所有内容了。