---
layout:     post
title:      nginx完整示例配置详解
subtitle:   
date:       2019-03-21
author:     yandenghong
header-img: img/post-bg-supercar.jpg
catalog: true
tags:
    - nginx
---

## 前言
最近花了一个星期的空闲时间整理出了这篇文章，主要目的是为了在配置nginx时作为全面的参考，也是为了方便一些在配置nginx时对某些参数感到疑惑的人群。
因为篇幅较长，如果你想查询某个参数，你只需在本页`Ctrl + F`然后输入参数名即可，若没有查询到可以以评论或者邮件的方式告知我，我来添加。

## nginx.conf

### 全局配置

#### user
```text
Syntax:	user user [group];
Default: user nobody nobody;
```
> 如果你的服务器是windows，则可以忽略这项配置。

定义工作进程使用的用户和组。如果省略group，则组名称和用户名一致。
如果没有USER指令,默认为nginx使用的用户。可以用`ps aux | grep nginx`来查看nginx的用户.
如果是root，那对应的用户组也是root。可以这样指定`user root;`。
#### worker_processes
作者的原话

As a general rule you need the only worker with large number of worker_connections, say 10,000 or 20,000.
However, if nginx does CPU-intensive work as SSL or gzipping and you have 2 or more CPU, then you may set worker_processes to be equal
to CPU number.Besides, if you serve many static files and the total size of the files is bigger than memory, then you may increase worker_processes to utilize a full disk bandwidth.

Igor Sysoev

意思很明确，一般情况下设置为`1`就够用了，如果你的服务器有多个cpu，则设置为`cpu个数`。此外如果你提供许多静态文件并且文件的总大小大于内存，那么可以增加worker_processes以使用完整的磁盘带宽。
> 注意:这里的cpu个数指的是逻辑cpu个数,不是物理cpu个数。正常情况下，逻辑cpu个数=物理cpu个数 * 每颗物理cpu的核数。如果你的cpu支持超线程（Intel的超线程技术，可以在逻辑上再分一倍的cpu core出来），逻辑cpu个数= 物理CPU个数 * 每颗物理CPU的核数 * 2。

linux下的:
* 查询CPU个数：`cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l`

* 查询核数：`cat /proc/cpuinfo| grep "cpu cores"| uniq`

* 查询逻辑CPU总数：`cat /proc/cpuinfo| grep "processor"| wc -l`

如果`逻辑cpu总数`不等于`cpu个数 * 核数`，　那说明你的cpu是支持超线程技术的。

#### error_log
设置错误日志路径并指定级别(`debug` | `info` | `notice` | `warn` | `error` | `crit`)
可以放在Main块中全局配置，也可以放在不同的虚拟主机中单独记录虚拟主机的错误信息。根据你的实际需求决定。
```text
error_log    <FILE>    <LEVEL>
关键字        日志文件   错误日志级别
```
* 关键字：其中关键字error_log不能改变。

* 日志文件：可以指定任意存放日志的目录。

* 错误日志级别：常见的错误日志级别有[debug | info | notice | warn | error | crit | alert | emerg]，级别越高记录的信息越少。

* 生产场景一般是 `warn`, `error`, `crit`这三个级别之一。

> 注意: 不要配置`info`等等级较低的级别，会带来大量的磁盘I/O消耗。

#### pid
设置nginx的pid文件存放目录。

#### worker_rlimit_nofile
同时连接的数量受限于系统上可用的文件描述符的数量，因为每个套接字将打开一个文件描述符。
一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（系统的值`ulimit -n`）与nginx进程数相除，但是nginx分配请求并不均匀，所以建议与`ulimit -n`的值保持一致。

### events块配置
该部分用来配置工作模式与连接数上限。
#### accept_mutex
```text
accept_mutex on;
```
设置网络连接序列化，防止惊群现象发生，默认为`on`。
> 惊群现象: 一个网络连接到来，多个睡眠的进程被同时叫醒，但只有一个进程能获得连接，这样会影响系统性能。

#### multi_accept
```text
multi_accept on;
```
设置一个进程是否同时接受多个网络连接，默认为`off`。默认为一个工作进程只能一次接受一个新的网络连接。

#### use
设置事件驱动模型, 可选模型:`select`|`poll`|`kqueue`|`epoll`|`resig`|`/dev/poll`|`eventport`。

标准事件模型: 

`select`,`poll`。如果当前系统不存在更有效的方法，nginx会选择其中之一。

高效事件模型: 

`kqueue`:使用于FreeBSD 4.1+, OpenBSD 2.9+, NetBSD 2.0 和 MacOS X.使用双处理器的MacOS X系统使用kqueue可能会造成内核崩溃。
 
`epoll`: 使用于Linux内核2.6版本及以后的系统。

`/dev/poll`: 使用于Solaris 7 11/99+, HP/UX 11.22+ (eventport), IRIX 6.5.15+ 和 Tru64 UNIX 5.1A+。

`eventport`: 使用于Solaris 10. 为了防止出现内核崩溃的问题， 有必要安装安全补丁。

> 查看linux版本号可以使用`cat /proc/version`命令。

#### worker_connections
```text
syntax:worker_connections number;
default: 1024
context:events
```
工作进程的最大连接数量。

如果把nginx能够处理的最大连接数用max_clients来表示,则理论上：

nginx作为http服务器的时候：
    
    max_clients = worker_processes * worker_connections
nginx作为反向代理服务器的时候：
    
    max_clients = worker_processes * worker_connections/4

因为并发受IO约束，max_clients的值须小于系统可以打开的最大文件数。而系统可以打开的最大文件数和内存大小成正比。
> linux查看系统可以打开的最大文件数:`cat /proc/sys/fs/file-max`。

综上, worker_connections 的值需根据 worker_processes 进程数目和系统可以打开的最大文件总数进行适当地进行设置
，使得并发总数小于操作系统可以打开的最大文件数目。其实质也就是根据主机的物理CPU和内存进行配置。

#### keepalive_timeout
keepalive超时时间。 这里指的是http层面的keep-alive 并非tcp的keepalive。

#### client_header_buffer_size
客户端请求头部的缓冲区大小。这个可以根据你的系统分页大小来设置，一般一个请求头的大小不会超过1k，而通常系统分页都要大于1k，所以这里设置为系统分页大小。
> linux查看系统分页大小(单位是字节): `getconf PAGESIZE`

### http块设置

#### include 
引入一些单独的配置文件。

使用举例:
* `include    conf/mime.types;`: 文件扩展名与文件类型映射表。
* `include    /etc/nginx/proxy.conf;`: 设置代理配置。
* `include    /etc/nginx/fastcgi.conf;`: FastCGI相关参数是为了改善网站的性能：减少资源占用，提高访问速度。

#### index 
该指令可以在http块内全局设置，也可为每个server单独设置。用于指定主页。
```text
index    index.html index.htm index.php;
```

#### default_type
设置默认文件类型。
```text
default_type application/octet-stream;
```

#### log_format
设置日志格式。
```text
log_format   main '$remote_addr - $remote_user [$time_local]  $status '
    '"$request" $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
```

#### access_log
设置访问日志的存放目录。后面可以跟一个参数为日志的格式(在`log_format`中定义的,上面定义了`main`)。
该日志记录了哪些用户，哪些页面以及用户浏览器、ip和其他的访问信息。
```text
access_log   logs/access.log  main;
```

#### sendfile
设置为`on`表示启动高效传输文件的模式。`sendfile`可以让Nginx在传输文件时直接在磁盘和tcp socket之间传输数据。如果这个参数不开启，会先在用户空间（Nginx进程空间）申请一个buffer，用read函数把数据从磁盘读到cache，再从cache读取到用户空间的buffer，再用write函数把数据从用户空间的buffer写入到内核的buffer，最后到tcp socket。开启这个参数后可以让数据不用经过用户buffer。

#### tcp_nopush
```text
syntax: tcp_nopush on | off;

default:  tcp_nopush on;

context: http, server, location
```
该选项仅在使用`sendfile`的时候才开启。

#### server_names_hash_bucket_size
当配置多个虚拟主机时，该指令必须设置。否则默认的大小无法正常存储多个`server_name`。

nginx wiki:

保存服务器名字的hash表是由指令`server_names_hash_max_size`和`server_names_hash_bucket_size`所控制的。参数`hash bucket size`总是等于hash表的大小，并且是一路处理器缓存大小的倍数。在减少了在内存中的存取次数后，使在处理器中加速查找hash表键值成为可能。如果`hash bucket size`等于一路处理器缓存的大小，那么在查找键的时候，最坏的情况下在内存中查找的次数为2。第一次是确定存储单元的地址，第二次是在存储单元中查找键 值。因此，如果Nginx给出需要增大`hash max size`或`hash bucket size`的提示，那么首要的是增大前一个参数的大小.

#### listen
在`server`块中设置。指定监听的端口。如果没有设置该指令，则nginx会监听默认的80端口。

#### server_name
域名，可以有多个，可以用精确的域名，通配符，正则表达式来定义。

通配符名称可能仅在名称的开头或结尾包含星号。名称`www.*.example.org`和`w*.example.org`无效。但是，可以使用正则表达式指定这些名称，例如`~^www\..+\.example\.org$`和`~^w.*\.example\.org$`。星号可以匹配多个名称部分。名称`*.example.org`不仅匹配，`www.example.org`而且`www.sub.example.org`也匹配 。
`.example.org`形式的特殊通配符名称可用于匹配确切名称`example.org`和通配符名称`*.example.org`。

nginx使用的正则表达式与Perl编程语言（PCRE）使用的正则表达式兼容。要使用正则表达式，服务器名称必须以波形符号开头：

`server_name~ ^ www \ d + \ .example \ .net $;`

否则它将被视为一个确切的名称，或者如果表达式包含星号，则视为通配符名称（并且很可能是无效的名称）。
别忘了设置`^`和`$`锚点。它们在语法上不是必需的，但在逻辑上是必需的。另请注意，域名点应使用反斜杠进行转义。

另外，关于域名的详细配置请看[nginx是如何处理请求的][1], 负载均衡配置请看我的[nginx内置的4种负载均衡算法][2]。

#### location中的proxy_pass反向代理的小说明
`proxy_pass`后面的url加`/`，表示绝对根路径,则nginx不会把`location`中匹配的路径部分代理走;如果没有`/`，则会把匹配的路径部分也给代理走。

例子:

不带`/`
```text
location /example/
{
    proxy_pass http://127.0.0.1:10080;
}
```
带`/`
```text
location /example/
{
    proxy_pass http://127.0.0.1:10080/;
}
```
针对不带`/`，如果访问url是`http://server/example/test.jsp`，则被nginx代理后，请求路径会变成`http://proxy_pass/example/test.jsp`，将`example/`作为根路径，请求`example/`路径下的资源

针对带`/`，如果访问url是`http://server/example/test.jsp`，则被nginx代理后，请求路径会变为`http://proxy_pass/test.jsp`，直接访问server的根路径。

#### 图片缓存时间设置
```text
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
{
    expires 10d;
}
```

#### JS和CSS缓存时间设置
```text
location ~ .*\.(js|css)?$
{
    expires 1h;
}
```

#### gzip设置
在http块中设置。
```text
gzip on; # 开启gzip压缩输出
gzip_min_length 1k; # 最小压缩文件大小
gzip_buffers 4 16k; # 压缩缓冲区
gzip_http_version 1.0; # 识别http的协议版本(1.0/1.1)
gzip_comp_level 2; # 1压缩比最小处理速度最快，9压缩比最大但处理速度最慢(传输快但比较消耗cpu)
# 匹配mime类型进行压缩，无论是否指定,”text/html”类型总是会被压缩的。
gzip_types text/plain application/x-javascript text/css application/xml;
gzip_vary on; # 和http头有关系，加个vary头，给代理服务器用的，有的浏览器支持压缩，有的不支持，所以避免浪费不支持的也压缩，所以根据客户端的HTTP头来判断，是否需要压缩
```




```text
user       www www;  ## Default: nobody
worker_processes  5;  ## Default: 1
error_log  logs/error.log;
pid        logs/nginx.pid;
worker_rlimit_nofile 8192;

events {
  worker_connections  4096;  ## Default: 1024
}

http {
  include    conf/mime.types;
  include    /etc/nginx/proxy.conf;
  include    /etc/nginx/fastcgi.conf;
  index    index.html index.htm index.php;

  default_type application/octet-stream;
  log_format   main '$remote_addr - $remote_user [$time_local]  $status '
    '"$request" $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
  access_log   logs/access.log  main;
  sendfile     on;
  tcp_nopush   on;
  server_names_hash_bucket_size 128; # this seems to be required for some vhosts

  server { # php/fastcgi
    listen       80;
    server_name  domain1.com www.domain1.com;
    access_log   logs/domain1.access.log  main;
    root         html;

    location ~ \.php$ {
      fastcgi_pass   127.0.0.1:1025;
    }
  }

  server { # simple reverse-proxy
    listen       80;
    server_name  domain2.com www.domain2.com;
    access_log   logs/domain2.access.log  main;

    # serve static files
    location ~ ^/(images|javascript|js|css|flash|media|static)/  {
      root    /var/www/virtual/big.server.com/htdocs;
      expires 30d;
    }

    # pass requests for dynamic content to rails/turbogears/zope, et al
    location / {
      proxy_pass      http://127.0.0.1:8080;
    }
  }

  upstream big_server_com {
    server 127.0.0.3:8000 weight=5;
    server 127.0.0.3:8001 weight=5;
    server 192.168.0.1:8000;
    server 192.168.0.1:8001;
  }

  server { # simple load balancing
    listen          80;
    server_name     big.server.com;
    access_log      logs/big.server.access.log main;

    location / {
      proxy_pass      http://big_server_com;
    }
  }
}
```
## proxy.conf
* `proxy_redirect`: 当上游服务器返回的响应是重定向或刷新请求（如HTTP响应码是301或者302）时，`proxy_redirect`可以重设HTTP头部的location或refresh字段。

* `proxy_set_header`: 就是可设置请求头，并将头信息传递到服务器端。

* `client_max_body_size`: 设置nginx能处理的请求body大小。 如果请求大于指定的大小，则返回HTTP 413（Request Entity too large）错误。 如果服务器处理大文件上传，则该指令非常重要。默认情况下，该指令值为1m。

* `client_body_buffer_size`: 设置用于请求body的缓冲区大小。 如果body超过缓冲区大小，则完整body或其一部分将写入临时文件。 如果nginx配置为使用文件而不是内存缓冲区，则该指令会被忽略。 默认情况下，该指令为32位系统设置一个8k缓冲区，为64位系统设置一个16k缓冲区。 该指令在nginx配置的http，server和location区块使用。

* `proxy_connect_timeout`: 默认值为60秒，该指令设置与upstream server的连接超时时间。

* `proxy_send_timeout`: 默认值为60秒，该指令设置了发送请求给upstream服务器的超时时间，如果超时后，upstream没有收到新的数据，nginx会关闭连接。

* `proxy_read_timeout`: 默认值为60秒，该指令设置与代理服务器的读超时时间。它决定了nginx会等待多长时间来获得请求的响应。这个时间不是获得整个response的时间，而是两次reading操作的时间。

* `proxy_buffers`: 设置缓冲区的大小和数量，从被代理的后端服务器取得的响应内容，会放置到这里。 默认情况下，一个缓冲区的大小等于内存页面大小，可能是4K也可能是8K，这取决于服务器。

```text
proxy_redirect          off;
proxy_set_header        Host            $host;
proxy_set_header        X-Real-IP       $remote_addr;
proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
client_max_body_size    10m;
client_body_buffer_size 128k;
proxy_connect_timeout   90;
proxy_send_timeout      90;
proxy_read_timeout      90;
proxy_buffers           32 4k;
```

## fastcgi.conf
```text
fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
fastcgi_param  QUERY_STRING       $query_string;
fastcgi_param  REQUEST_METHOD     $request_method;
fastcgi_param  CONTENT_TYPE       $content_type;
fastcgi_param  CONTENT_LENGTH     $content_length;
fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
fastcgi_param  REQUEST_URI        $request_uri;
fastcgi_param  DOCUMENT_URI       $document_uri;
fastcgi_param  DOCUMENT_ROOT      $document_root;
fastcgi_param  SERVER_PROTOCOL    $server_protocol;
fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;
fastcgi_param  REMOTE_ADDR        $remote_addr;
fastcgi_param  REMOTE_PORT        $remote_port;
fastcgi_param  SERVER_ADDR        $server_addr;
fastcgi_param  SERVER_PORT        $server_port;
fastcgi_param  SERVER_NAME        $server_name;

fastcgi_index  index.php;

fastcgi_param  REDIRECT_STATUS    200;
```

## mime.types
```text
types {
  text/html                             html htm shtml;
  text/css                              css;
  text/xml                              xml rss;
  image/gif                             gif;
  image/jpeg                            jpeg jpg;
  application/x-javascript              js;
  text/plain                            txt;
  text/x-component                      htc;
  text/mathml                           mml;
  image/png                             png;
  image/x-icon                          ico;
  image/x-jng                           jng;
  image/vnd.wap.wbmp                    wbmp;
  application/java-archive              jar war ear;
  application/mac-binhex40              hqx;
  application/pdf                       pdf;
  application/x-cocoa                   cco;
  application/x-java-archive-diff       jardiff;
  application/x-java-jnlp-file          jnlp;
  application/x-makeself                run;
  application/x-perl                    pl pm;
  application/x-pilot                   prc pdb;
  application/x-rar-compressed          rar;
  application/x-redhat-package-manager  rpm;
  application/x-sea                     sea;
  application/x-shockwave-flash         swf;
  application/x-stuffit                 sit;
  application/x-tcl                     tcl tk;
  application/x-x509-ca-cert            der pem crt;
  application/x-xpinstall               xpi;
  application/zip                       zip;
  application/octet-stream              deb;
  application/octet-stream              bin exe dll;
  application/octet-stream              dmg;
  application/octet-stream              eot;
  application/octet-stream              iso img;
  application/octet-stream              msi msp msm;
  audio/mpeg                            mp3;
  audio/x-realaudio                     ra;
  video/mpeg                            mpeg mpg;
  video/quicktime                       mov;
  video/x-flv                           flv;
  video/x-msvideo                       avi;
  video/x-ms-wmv                        wmv;
  video/x-ms-asf                        asx asf;
  video/x-mng                           mng;
}
```

[1]: https://yandenghong.github.io/2019/03/13/how_nginx_request/
[2]: https://yandenghong.github.io/2019/03/15/nginx_load_balancing/
