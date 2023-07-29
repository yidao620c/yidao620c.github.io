---
title: Django1.9开发博客10- 全文搜索
date: 2015-08-21 15:32:28 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

Django本身不提供全文检索的功能，但django-haystack为其提供了全文检索的框架。
django-haystack能为Django提供whoosh,solr,Xapian和Elasticsearc四种全文检索引擎作为后端。
其中whoosh为纯python的实现，不是非常大型的应用，是没有问题的。
本文将介绍Django1.9中通过django-haystack与whoosh集成以及whoosh的中文支持。
<!-- more -->

### 安装依赖：

```bash
pip install django-haystack
pip install whoosh
pip install jieba
```

### 建立模型

我们以文章为搜索目标，现在我的app名字为blog，
模型文件是mysite/blog/models.py ：

```python
# coding=utf-8
from django.db import models
@python_2_unicode_compatible
class Post(models.Model):
    class Meta:
        verbose_name = u'文章'
        verbose_name_plural = u'文章'
    # 作者
    author = models.ForeignKey(User)
    # 标题
    title = models.CharField(max_length=200)
    # 正文
    text = models.TextField()
    # 标签
    tags = models.ManyToManyField(Tag)
    # 分类目录
    category = models.ForeignKey(Category)
    # 点击量
    click = models.IntegerField(default=0)
    # 创建时间
    created_date = models.DateTimeField(default=timezone.now)
    # 发布时间
    published_date = models.DateTimeField(blank=True, null=True)

    def publish(self):
        self.published_date = timezone.now()
        self.save()

    def __str__(self):
        return self.title
```

### search_indexes.py

在app目录下建立一个search_indexes.py（mysite/blog/search_indexes.py）代码如下：

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-
from models import Post
from haystack import indexes
class PostIndex(indexes.SearchIndex, indexes.Indexable):
    # 文章内容
    text = indexes.CharField(document=True, use_template=True)
    # 对title字段进行索引
    title = indexes.CharField(model_attr='title')
    def get_model(self):
        return Post

    def index_queryset(self, using=None):
        return self.get_model().objects.all()
```

*备注*：search_indexes.py文件名不能修改，否则报错：`No fields were found in any search_indexes.`

### post_text.txt

因为在search_indexes.py使用了use_template=True，所以可以同时使用模板对索引字段进行定义。

如：`mysite/blog/templates/search/indexes/blog/post_text.txt`:

```
@@ object.title @@
@@ object.text @@
```

### settings.py

```python
# Application definition
INSTALLED_APPS = (
    ...
    'haystack',
)
```

### urls.py

```python
urlpatterns = patterns(
    '',
    url(r'^admin/', include(admin.site.urls)),
    url(r'^xadmin/', include(xadmin.site.urls), name='xadmin'),
    url(r'^accounts/login/$', 'django.contrib.auth.views.login'),
    url(r'^accounts/logout/$', 'django.contrib.auth.views.logout', {'next_page': '/'}),
    url(r'^search/', include('haystack.urls')),
    url(r'', include('blog.urls')),
)
```

### jieba中文分词

jieba其实已经提供了集成whoosh的ChineseAnalyzer，
也就是说不需要自己写ChineseAnalyzer了，直接在whoosh_backend.py中直接引用就好；
同时，不推荐将whoosh_backend.py放到Lib下面，这样移植性会有问题，自己的代码，还是放在项目下面为妙。

1\. 将文件haystack.backends.whoosh_backend.py拷贝到app下面，并重命名为whoosh_cn_backend.py，
如blog/whoosh_cn_backend.py。重点的改造有：

* 增加：

```python
from jieba.analyse import ChineseAnalyzer
```

* 修改

```python
schema_fields[field_class.index_fieldname] = TEXT(stored=True, analyzer=ChineseAnalyzer(),
                                                  field_boost=field_class.boost, sortable=True)
```

2\. 修改后端引擎，setting.py配置：

```python
# full text search
HAYSTACK_CONNECTIONS = {
    'default': {
        'ENGINE': 'blog.whoosh_cn_backend.WhooshEngine',
        'PATH': os.path.join(BASE_DIR, 'whoosh_index'),
    },
}
```

### 重建索引

```python
python
manage.py
rebuild_index
```

### 索引更新

最简单的办法就是在settings.py中添加：

```python
HAYSTACK_SIGNAL_PROCESSOR = 'haystack.signals.RealtimeSignalProcessor'
```

### 自定义搜索示例

(1) 先定义view：

```python
from haystack.forms import SearchForm
def full_search(request):
    """全局搜索"""
    keywords = request.GET['q']
    sform = SearchForm(request.GET)
    posts = sform.search()
    return render(request, 'blog/post_search_list.html',
                  {'posts': posts, 'list_header': '关键字 \'{}\' 搜索结果'.format(keywords)})
```

(2) 然后在template页面中：

```html
<!-- searchbox START -->
<div id="searchbox">
    <form action="@% url 'blog.views.full_search' %@" method="get">
        <div class="content">
            <label>
                <input type="text" class="textfield searchtip" name="q" size="24" value="">
            </label>
            <input type="submit" class="button" value="">
        </div>
    </form>
</div>
```

更详细内容请参考官方文档：<http://django-haystack.readthedocs.org/en/latest/>
