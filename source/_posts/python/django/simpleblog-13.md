---
title: Django1.9开发博客13- redis缓存
date: 2015-08-25 20:27:55 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

Redis 是一个高性能的key-value数据库。redis的出现，
很大程度补偿了memcached这类keyvalue存储的不足，在部分场合可以对关系数据库起到很好的补充作用。
它提供了Python，Ruby，Erlang，PHP客户端，使用很方便。

目前Redis已经发布了3.0版本，正式支持分布式，这个特性太强大，以至于你再不用就对不住自己了。
<!-- more -->

### 性能测试

服务器配置：Linux 2.6, Xeon X3320 2.5Ghz

SET操作每秒钟110000次，GET操作每秒钟81000次

stackoverflow网站使用Redis做为缓存服务器。

### 安装redis

服务器安装篇我写了专门文章，
请参阅[redis入门与安装](http://yidao620c.github.io/2015/07/01/java/redis01.html)

### django中的配置

我们希望在本博客系统中，对于文章点击数、阅览数等数据实现缓存，提高效率。

#### requirements.txt

添加如下内容，方便以后安装软件依赖，由于在pythonanywhere上面并不能安装redis服务，所以本章只能在本地测试。

```
redis==2.10.5
django-redis==4.4.2
APScheduler==3.1.0
```

#### settings.py配置

新增内容

```python
from urllib.parse import urlparse
import dj_database_url

redis_url = urlparse(os.environ.get('REDISTOGO_URL', 'redis://localhost:6959'))
CACHES = {
    'default': {
        'BACKEND': 'redis_cache.cache.RedisCache',
        'LOCATION': '{0}:{1}'.format(redis_url.hostname, redis_url.port),
        'OPTIONS': {
            'DB': 0,
            'PASSWORD': redis_url.password,
            'CLIENT_CLASS': 'redis_cache.client.DefaultClient',
            'PICKLE_VERSION': -1,  # Use the latest protocol version
            'SOCKET_TIMEOUT': 60,  # in seconds
            'IGNORE_EXCEPTIONS': True,
        }
    }
}

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'

# 本地开发配置放在local_settings.py中
try:
    from .local_settings import *
except ImportError:
    pass
```

#### local_settings.py配置

这个是本地开发时候使用到的配置文件

```python
DEBUG = True

CACHES = {
    'default': {
        'BACKEND': 'redis_cache.cache.RedisCache',
        'LOCATION': '192.168.203.95:6379:1',
        'OPTIONS': {
            'CLIENT_CLASS': 'redis_cache.client.DefaultClient',
            # 'PASSWORD': 'secretpassword',
            'PICKLE_VERSION': -1,  # Use the latest protocol version
            'SOCKET_TIMEOUT': 60,  # in seconds
            'IGNORE_EXCEPTIONS': True,
        }
    }
}
```

### 使用方法

#### cache_manager.py缓存管理器

我们新建一个缓存管理器cache_manager.py，内容如下

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
"""
Topic: redis缓存管理器
"""
from ..models import Post
from redis_cache import get_redis_connection
from apscheduler.schedulers.background import BackgroundScheduler

RUNNING_TIMER = False
REDIS_DB = get_redis_connection('default')


def update_click(post):
    """ 更新点击数 """
    if REDIS_DB.hexists("CLICKS", post.id):
        print('REDIS_DB.hexists...' + str(post.id))
        REDIS_DB.hincrby('CLICKS', post.id)
    else:
        print('REDIS_DB.not_hexists...' + str(post.id))
        REDIS_DB.hset('CLICKS', post.id, post.click + 1)
    run_timer()


def get_click(post):
    """ 获取点击数 """
    if REDIS_DB.hexists("CLICKS", post.id):
        return REDIS_DB.hget('CLICKS', post.id)
    else:
        REDIS_DB.hset('CLICKS', post.id, post.click)
        return post.click


def sync_click():
    """同步文章点击数"""
    print('同步文章点击数start....')
    for k in REDIS_DB.hkeys('CLICKS'):
        try:
            p = Post.objects.get(k)
            print('db_click={0}'.format(p.click))
            cache_click = get_click(p.id)
            print('cache_click={0}'.format(cache_click))
            if cache_click != p.click:
                p.click = get_click(p.id)
                p.save()
        except:
            pass
```

#### views.py修改

然后我们修改view.py，在相应的action里使用这个cache_manager：

```python
from .commons import cache_manager


def post_list(request):
    """所有已发布文章"""
    posts = Post.objects.annotate(num_comment=Count('comment')).filter(
        published_date__isnull=False).prefetch_related(
        'category').prefetch_related('tags').order_by('-published_date')
    for p in posts:
        p.click = cache_manager.get_click(p)
    return render(request, 'blog/post_list.html', {'posts': posts})


def post_detail(request, pk):
    try:
        pass
    except:
        raise Http404()
    if post.published_date:
        cache_manager.update_click(post)
        post.click = cache_manager.get_click(post)
```

其他的我就不多演示了。

