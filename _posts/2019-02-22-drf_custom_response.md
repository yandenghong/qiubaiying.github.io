---
layout:     post
title:      django rest framework构建RestAPI的高可用自定义返回格式
subtitle:   
date:       2019-02-22
author:     yandenghong
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - django rest framework
    - python
    - api
---

## 前言

使用过django rest framework的同学们想必一定遇到过其默认返回格式不符合项目要求的问题。本篇blog就是为你来解决这个问题而生的！

## 正文

比如我们现在想要的返回格式是这样的：

```python
{
   "data":[],
   "code":200,
   "msg": ""
}
```

框架自带的Response定义其返回，要修改返回，就要从这个Reponse类入手，话不多说，直接上代码:

```python
from django.utils import six
from rest_framework.response import Response
from rest_framework.serializers import Serializer


class JsonResponse(Response):
    """
    An HttpResponse that allows its data to be rendered into
    arbitrary media types.
    """

    def __init__(self, data=None, code=None, msg=None,
                 status=None,
                 template_name=None, headers=None,
                 exception=False, content_type=None):
        """
        Alters the init arguments slightly.
        For example, drop 'template_name', and instead use 'data'.
        Setting 'renderer' and 'media_type' will typically be deferred,
        For example being set automatically by the `APIView`.
        """
        super(Response, self).__init__(None, status=status)

        if isinstance(data, Serializer):
            msg = (
                'You passed a Serializer instance as data, but '
                'probably meant to pass serialized `.data` or '
                '`.error`. representation.'
            )
            raise AssertionError(msg)
        # 这里定义格式
        self.data = {"code": code, "msg": msg, "data": data}
        
        self.template_name = template_name
        self.exception = exception
        self.content_type = content_type

        if headers:
            for name, value in six.iteritems(headers):
                self[name] = value

```

这样继承并修改后，在api view中使用我们自己的JsonResponse即可。

现在又有一个新的需求，有些接口返回的数据条数很多，需要分页，想要的格式是这样的:
```python
{    
    "count": 21387,
    "next": "http://www.xxxx.xxxx",
    "previous": "http://www.xxxx.xxxx",
    "data": [],
    "msg": "msg",
    "code": 200
}
```

也就是说我们要在原来的格式中加入分页，并且注意需求：是"有些"接口， 因此我们还要兼容原来的格式,分页器的话，没写过的可以自己写写锻炼一下，我呢是直接使用drf自带的LimitOffsetPagination。

```python
from django.utils import six
from rest_framework.response import Response
from rest_framework.serializers import Serializer


class JsonResponse(Response):
    """
    An HttpResponse that allows its data to be rendered into
    arbitrary media types.
    """
    
    # 注意这里的初始化参数，我新增了一个paginator参数，就是分页器的实例
    def __init__(self, data=None, code=None, msg=None,
                 status=None,
                 template_name=None, headers=None,
                 exception=False, content_type=None, paginator=None):
        """

        Alters the init arguments slightly.
        For example, drop 'template_name', and instead use 'data'.
        Setting 'renderer' and 'media_type' will typically be deferred,
        For example being set automatically by the `APIView`.
        """
        super(Response, self).__init__(None, status=status)

        if isinstance(data, Serializer):
            msg = (
                'You passed a Serializer instance as data, but '
                'probably meant to pass serialized `.data` or '
                '`.error`. representation.'
            )
            raise AssertionError(msg)
        # 之所以加上判断是为了兼容第一种格式，在没有分页器传入的时候，可以正常返回
        if not paginator:
            self.data = {"status": code, "msg": msg, "data": data}
        else:
            # 这里的分页器实例调用的方法都是drf已经为我们封装好的方法
            self.data = {"count": paginator.count,
                         "next": paginator.get_next_link(),
                         "previous": paginator.get_previous_link(),
                         "data": data,
                         "msg": msg,
                         "status": code
                         }
        self.template_name = template_name
        self.exception = exception
        self.content_type = content_type

        if headers:
            for name, value in six.iteritems(headers):
                self[name] = value
```

在经过上面的改造后，我们在view中应该这样使用:
```python
from rest_framework import status
from rest_framework.pagination import LimitOffsetPagination
from rest_framework.views import APIView
from .custom_response import JsonResponse

class DemoData(APIView):
    """示例APIView"""

    def get(self, request):
        # 这里是纯作示例的查询参数序列化器, 因此没有import, 作用是对查询参数进行校验
        serializer = DemoSerializer(data=request.query_params)
        # 分页器实例化
        page = LimitOffsetPagination()
        if serializer.is_valid():
            # 对数据进行分页
            paged_data = page.paginate_queryset(queryset=data.order_by('id'), request=request, view=self)
            # 这里是纯作示例的结果数据序列化器，因此没有import, 作用是对查询结果进行json序列化
            data = DemoResultSerializer(paged_data, many=True).data
            # 上面的分页器实例在这里以paginator参数传入
            return JsonResponse(data=data, code=200, msg="查询成功", paginator=page,
                                status=status.HTTP_200_OK)

        else:
            return JsonResponse(code=400, msg=serializer.errors, status=status.HTTP_400_BAD_REQUEST)

```

经过这些步骤，即实现了高可用的自定义API返回格式.
