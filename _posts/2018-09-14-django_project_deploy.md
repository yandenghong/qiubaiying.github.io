---
layout:     post
title:      nginx+uwsgi+django+celery部署日志
subtitle:   包含了详细的配置与相应说明
date:       2018-09-14
author:     yandenghong
header-img: img/post-moon.png
catalog: true
tags:
    - nginx
    - uwsgi
    - Django
    - 部署
    - celery
---

## 前言
为什么要选择nginx + uwsgi + django?

首先，如果你对django很了解或者很熟悉, 那你应该知道,django提供的服务器是仅供开发调试用的,它是单进程的,而且,在debug模式关闭之后,
它就不处理静态文件的请求了,这样的结果就是网站的所有静态文件请求全部是404.你可能会说,我的网站访问量不会很多, 就用django自带的单进程服务器
也够用, 我不关闭debug不就好了.然而,这样是不合理的!因为在debug开启的时候,django会把所有的sql执行结果放在内存里,也就是说,有可能会导致内存溢出.
所以, 必须要用专业的服务器来部署django项目.

Nginx是俄罗斯大牛Igor Sysoev编写的免费开源的面向性能的异步的web服务器.使用的内存比 Apache 少得多,每秒可以处理大约四倍于 Apache 的请求.
同时也可以用作反向代理,负载均衡和HTTP缓存.大部分的网站都使用它来处理网站的静态文件.之前有亲眼看到一个网站某个页面，加载完耗时4秒多, 在用nginx
做静态文件处理后,加载时长缩短到了300~400ms.整整十倍,所以,用它来处理静态文件是极好的.

## 正文
### 环境准备
uwsgi安装(记得添加到你的项目的requirements中)
```
pip install uwsgi
```
uwsgi配置文件(文件格式：.ini,我的参数设置只有一部分,这已经够用了,如果不满足你的需求,详细配置上这看：https://uwsgi-docs-zh.readthedocs.io/zh_CN/latest/)
```
[uwsgi] # 这个必须要有
# 项目在服务器中的目录(绝对路径)
chdir = 你的项目目录

#是否使用主线程
master = true

# 虚拟环境目录(绝对路径)
home = 你的虚拟环境目录

# pid 文件路径
pidfile = 你的项目路径/uwsgi.pid

# Django's wsgi 文件目录(通过django-admin 命令生成的项目目录都会有一个和项目容器名相同的目录,在这个目录下有wsgi.py)
wsgi-file = 项目名/wsgi.py

# 排队请求数 可以理解为最高并发量
listen = 范围是0 - 65000

# 最大进程数
processes = 2


# 与外界连接的端口号, Nginx通过这个端口转发给uWSGI
socket = :端口号

#目录下文件改动时自动重启
touch-reload = 目录

#Python文件改动时自动重启
#py-auto-reload = 1

# 当服务器退出的时候自动清理环境
vacuum = true

# 后台运行并把日志存到.log文件
daemonize = 你的日志目录/uwsgi.log

# 日志大小，当大于这个大小会进行切分 (Byte)
log-maxsize = 50000000(按需设置,我设置了50MB)

```
### 静态文件收集

django settings中的STATICFILES_DIRS这个目录主要是存放开发时的静态文件，当debug关闭后,虽然还可以用，但是它里面没有admin 的静态文件，因此需要
设置STATIC_ROOT路径, 这个路径就是执行下面这个命令之后,所有静态文件存放的目录，如果你觉得线上有两个静态目录比较累赘，可以把STATICFILES_DIRS注释掉,
STATIC_ROOT路径设置为原来的路径就可以了，然后就可以在线上执行该命令了。这个目录，也就是待会你要在nginx中配置的静态文件目录.
```
python manage.py collectstatic
```

### uwsgi启动
```
uwsgi --ini 你的uwsgi配置文件名(.ini格式)
```
成功启动可以在日志文件里看到worker,设置了几个进程，就会有几个worker.

### celery启动
```
nohup celery -A 你的celery配置所在目录名 worker -l info --logfile 日志目录/celery.log &

nohup celery -A 你的celery配置所在目录名 beat -l info  --logfile 日志目录/celery_beat.log &
```

### celery 重启(其实可以放在uwsgi里，但很多时候，celery跟django的重启并不同步，因此我自己写了一个重启脚本)
```python
"""
此脚本仅用于celery重启.
"""
import os


class RestartBot:
    _os = os
    _celery_keyword_command = "ps aux|grep celery|grep -v grep|awk '{print $2}'"
    _start_celery_worker = "上面celery启动的第一个命令"
    _start_celery_beat = "上面celery启动的第二个命令"

    def _get_pid_list_by_execute_filter(self, command_str):
        """利用os执行过滤命令，返回pid 的list"""
        cleaned_pid_list = list()
        raw_pid_list = self._os.popen(command_str).readlines()
        if raw_pid_list:
            for pid in raw_pid_list:
                cleaned_pid = pid.strip("\n")
                cleaned_pid_list.append(cleaned_pid)
        return cleaned_pid_list

    @staticmethod
    def _build_up_kill_command(pid_list):
        """拼接kill命令 返回str"""
        init_command = "kill "
        for pid in pid_list:
            init_command += pid
            init_command += " "
        return init_command

    def kill_process(self, pid_list):
        """执行kill命令"""
        kill_command = self._build_up_kill_command(pid_list)
        self._os.system(kill_command)

    def execute(self, command):
        self._os.system(command)

    def main(self, beat=False):
        """重启流程控制"""
        # 获取celery pid list
        print("============>开始获取celery 进程")
        celery_pid_list = self._get_pid_list_by_execute_filter(self._celery_keyword_command)
        print("============>获取到celery 进程:{0}".format(celery_pid_list))
        # kill
        print("============>开始清理celery 进程")
        self.kill_process(celery_pid_list)
        print("============>celery 进程清理完毕")
        # 启动celery
        print("============>开始启动celery worker")
        self.execute(self._start_celery_worker)
        print("============>celery worker启动完毕")
        if beat:
            print("============>开始启动celery beat")
            self.execute(self._start_celery_beat)
            print("============>celery beat 启动完毕")
        print("============>重启成功")


if __name__ == '__main__':
    rb = RestartBot()
    rb.main(beat=True)
```
使用呢很简单,直接用python执行就好了.

### nginx 配置
```
server {
  listen 80;
  server_name 你的域名;
  autoindex off;
  client_max_body_size 20m;

  access_log 日志路径;
  error_log 日志路径;

  location / {
    allow  all;
    uwsgi_pass 127.0.0.1:你的服务的端口; # uwsgi和nginx通信就是通过这两个设置
    include uwsgi_params;
  }

  location /static/ {
    autoindex off;
    alias 上面收集的STATIC_ROOT的绝对路径;
    expires 缓存时长;
  }

}

server {
  listen   443 ssl;
  server_name 你的域名;
  autoindex off;
  client_max_body_size 20m;
  access_log 日志路径;
  error_log 日志路径;
  ssl_certificate crt文件路径;
  ssl_certificate_key key文件路径;

  location = /nginx-status {
    allow       127.0.0.1;
    deny        all;
    stub_status on;
  }

  location / {
    allow  all;
    uwsgi_pass 127.0.0.1:你的服务端口;
    include uwsgi_params;
  }

  location /static/ {
    autoindex off;
    alias 上面收集的STATIC_ROOT的绝对路径;
    expires 缓存时长;
  }

}

upstream django {
  server localhost:你的服务端口 weight=1;
}
```

### 重启nginx
```
nginx -s reload
```
## 重启完nginx后，你应该可以打开浏览器愉快的访问你的网站了~

### 附：uwsgi常用命令

停止
```
uwsgi --stop uwsgi.pid
```
重新加载
```
uwsgi --reload uwsgi.pid
```
