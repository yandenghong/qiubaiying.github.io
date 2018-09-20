---
layout:     post
title:      基于Twitter Webhook开发
subtitle:
date:       2018-09-20
author:     yandenghong
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Twitter
    - Webhook
---

## 前言
不得不说，Twitter的webhook接口使用步骤有点繁琐，不像Telegram，直接调一个接口设置完就可以使用。此篇记录了我如何成功使用Twitter webhook。

## 步骤一:开发者帐号申请(耗时最多)
想成为一名Twitter 开发者，首先你得是一名twitter用户，然后去https://developer.twitter.com这里申请开发者帐号，有一点需要注意的就是，
你将如何使用这个开发者帐号，用它来做什么， 一定要有个详细的，清楚的说明，因为这个是他们人工审核的，如果你随便填写，肯定不会通过申请的。填写完之后，
大概两到三天后会以邮件方式回复。

## 步骤二:APP创建，环境创建。
创建一个APP之后，如果你的webhook还需要能接收其他人发到你twitter的私信，那么你还要去修改一下APP权限，加一条direct messages，如下。

![](/img/twitter_permission.png)

> 图为：APP权限

然后，再去APP的Keys and tokens下面生成自己的Access token & Access token secret(Consumer key和Consumer secret key创建完APP就生成
了。)

环境创建：去Dev Environments下选择Account Activity API ,给环境起个好听又简洁的名字(比如dev)，然后添加刚才创建的app，环境就准备好了.

## 步骤三: CRC测试代码准备
要调用API去注册Webhook，首先要通过CRC测试，并且，在之后的使用过程中，twitter也会每小时来验证一次，如果验证不通过，那么这个接口就被Twitter
认为是无效的，也就不会再推送数据过来。如果你不知道这是什么，不用去twitter文档找，直接看下面的代码，比他们文档强多了。顺便吐槽一下，
他们文档的示例是python2的,都淘汰了，还不更新，真够懒的，接口文档也写的够烂。

```python
import hmac
import hashlib


def generate_signature(data):
    key_bytes = consumer_secret.encode('utf-8')
    data_bytes = data.encode('utf-8')
    return hmac.new(key_bytes, data_bytes, hashlib.sha256).digest()


class TwitterWebHookView(APIView):
    """twitter webhook api view"""
    def get(self, request):
        """twitter crc 验证"""
        # 由twitter传过来的token和你的消费者密钥生成的sha256 hash
        sha256_hash_digest = generate_signature(request.GET.get('crc_token'))

        # 将上面的hash进行base64编码之后按照指定响应格式组装
        response = {
            'response_token': ('sha256=' + (base64.b64encode(sha256_hash_digest).decode('utf-8')))
        }
        return JsonResponse(response)

    def post(self, request):
        """twitter account activity webhook api"""
        # 这里是接收webhook的接口
        pass
```

## 步骤四： 注册Webhook url
为了节省时间，我借用了第三方库：
```
pip install TwitterAPI
```
```python
import requests
from TwitterAPI import TwitterAPI

# 这里的key和secret不要填错了
twitter_api = TwitterAPI(consumer_key, consumer_secret, access_token, access_token_secret)

r = twitter_api.request('account_activity/all/:你的环境名/webhooks', {'url': 你的webhook url})
print(r.status_code)
print(r.text)
```
__注意__：这里注册成功后会返回webhook id，要记下来，后面订阅要使用.


## 步骤五：订阅account activity
这个库是有缺陷的，下面这个订阅，如果用它的twitter api去订阅，它一直发的是GET请求，而订阅是POST请求，issue我已经在github上提了。

所以我用了requests去自己发，认证依然用twitter_api的认证.
```python
r = requests.post(url='https://api.twitter.com/1.1/account_activity/all/development/subscriptions.json',
                  auth=twitter_api.auth,
                  data={'webhook_id': 上面的webhook id})
print(r.status_code)
print(r.text)
```
订阅成功后，状态码204, 没有任何响应内容。


## 步骤六: Account Events解析
订阅成功后，每当你的twitter帐号有什么事件，都会推送给你，
所有能推送的事件和每种事件的推送示例在这：https://developer.twitter.com/en/docs/accounts-and-users/subscribe-account-activity/guides/account-activity-data-objects#tweet_create_events

大功告成!
