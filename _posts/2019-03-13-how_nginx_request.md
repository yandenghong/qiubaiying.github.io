---
layout:     post
title:      nginx是如何处理请求的
subtitle:   本文翻译自nginx官网
date:       2019-03-13
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - nginx
---
### 原文链接
http://nginx.org/en/docs/http/request_processing.html
### 作者
**Igor Sysoev**

### 基于名称的服务器
当一个请求到达nginx后，nginx会先判断该由哪个服务器去处理该请求。
以下是一个简单的配置，三个服务器都监听80端口:

```text
server {
    listen      80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      80;
    server_name example.com www.example.com;
    ...
}
```
在此配置中，nginx仅测试请求头中的"Host"字段，以确定请求应路由到哪个服务器。
如果"Host"字段的值与任何服务器名称都不匹配，或者请求根本不包含这个字段，则nginx会将请求路由到此端口的默认服务器。

在上面的配置中，默认服务器是第一个(nginx的标准默认行为)。也可以使用listen指令中的default_server参数明确设置哪个服务器应该是默认的:
```text
server {
    listen      80 default_server;
    server_name example.net www.example.net;
    ...
}
```
> 注意:default_server是listen的属性，而不是server_name的属性。稍后会详细介绍。

### 如何防止未定义的server_name处理请求
请求头中没有"Host"字段的请求，应该被阻止，像下面这样定义一个只丢弃请求的服务器：

```text
server {
    listen      80;
    server_name "";
    return      444;
}
```
这里的配置中， server_name被设置为一个空字符串, 即可匹配到没有"HOST"请求头字段的请求， 并返回一个特殊的nginx的非标准代码444来关闭连接。

### 混合基于名称的服务器和基于IP的服务器
让我们看一个更复杂的配置，其中一些虚拟服务器监听不同的地址：
```text
server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      192.168.1.1:80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      192.168.1.2:80;
    server_name example.com www.example.com;
    ...
}

```
在此配置中，nginx首先根据每个server的listen指令测试请求的IP地址和端口。
然后，nginx找到与请求的IP和端口对应的server，用该server的server_name测试请求的“Host”头字段。
如果未找到server_name，则默认服务器将处理该请求。

举个例子，在192.168.1.1:80端口上收到的www.example.com请求将由192.168.1.1:80端口的默认服务器处理，也就是该地址的第一台服务器(没有指定default_server)，
因为该地址的server_name没有www.example.com。

如前所述，default_server是listen指令的属性，可以为不同的端口定义不同的默认服务器:
```text
server {
    listen      192.168.1.1:80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      192.168.1.1:80 default_server;
    server_name example.net www.example.net;
    ...
}

server {
    listen      192.168.1.2:80 default_server;
    server_name example.com www.example.com;
    ...
}

```
### 一个简单的PHP站点配置
现在让我们看一下nginx如何选择一个location来处理一个典型的简单PHP站点的请求：
```text
server {
    listen      80;
    server_name example.org www.example.org;
    root        /data/www;

    location / {
        index   index.html index.php;
    }

    location ~* \.(gif|jpg|png)$ {
        expires 30d;
    }

    location ~ \.php$ {
        fastcgi_pass  localhost:9000;
        fastcgi_param SCRIPT_FILENAME
                      $document_root$fastcgi_script_name;
        include       fastcgi_params;
    }
}

```
nginx首先搜索由文字字符串给出的最具体的prefix location，而不管列出的顺序如何。
**在以上的配置中唯一的前缀location是"/", 由于它匹配任何请求，它将被放在最后搜索。**
然后nginx按照配置文件中列出的顺序检查正则表达式构成的location。
第一个匹配表达式停止搜索，nginx将使用此location。如果没有正则表达式匹配请求，则nginx使用先前找到的最具体的前缀location。

需要注意的一点是，所有类型的location仅测试不带参数的请求行的URI部分。这样做是因为查询字符串中的参数可以通过多种方式给出，例如：
```text
/index.php?user=john&page=1
/index.php?page=1&user=john
```

此外，任何人都可以在查询字符串中请求任何内容:
```text
/index.php?page=1&something+else&user=john
```

现在让我们看看如何在上面的配置中处理请求：

* 请求`/logo.gif`首先与前缀location `/`匹配，然后由正则表达式`\.(gif|jpg|png)$`匹配，因此，它由后一个location处理。
* 请求`/index.php`也首先与前缀location `/`匹配，然后由正则表达式`\.(php)$`匹配。因此，它由后一个location处理，请求被传递给侦听localhost：9000的FastCGI服务器。 fastcgi_param指令将FastCGI参数SCRIPT_FILENAME设置为`/data/www/index.php`，FastCGI服务器执行该文件。变量$document_root等于root指令的值，变量$fastcgi_script_name等于请求URI，即`/index.php`。
* 请求`/about.html`仅与前缀location `/`匹配, 因此, 它由此location处理。使用指令"root /data/www"将请求映射到文件/data/www/about.html，并将文件发送到客户端。
* 处理请求`/`更复杂。它仅与前缀location `/`匹配，因此，它由此location处理。然后index指令根据其参数和`root /data/www`指令测试索引文件是否存在。如果文件/data/www/index.html不存在，并且文件/data/www/index.php存在，则指令执行内部重定向到`/index.php`，并且nginx再次搜索location, 就好像请求是由客户端发送的一样。正如我们之前看到的，重定向的请求最终将由FastCGI服务器处理。



