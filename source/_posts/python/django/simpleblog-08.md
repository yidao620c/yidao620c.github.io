---
title: Django1.9开发博客08- 继续完善
date: 2015-08-18 08:24:11 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

到现在为止我们已经完成的差不多了，并且基本的东西都已经学到了，是时候用起来了。
我们的博客还有很多功能需要完善，下面抛砖引玉新增几个功能，还有其他功能等你自己去发现和实现。
<!-- more -->

## 草稿箱

之前我们新建文章的时候只是是保存到数据库，也就是仅仅保存了草稿，还没有对外发布，
在博客首页上面是看不到的，因为published_date字段为空。这里我们需要添加一个草稿箱的链接。还是四部曲。

第一步，添加一个链接：

打开mysite/templates/mysite/base.html文件，在

```html
<h1><a href="/">Django Girls Blog</a></h1>
```

的上面一行添加如下链接：

```html
<a href="@% url 'post_draft_list' %@" class="top-menu">
<span class="glyphicon glyphicon-edit"></span></a>
```

第二步就是配置urls，在blog/urls.py中添加：

```python
url(r'^drafts/$', views.post_draft_list, name='post_draft_list'),
```

第三步在blog/views.py中添加一个view：

```python
def post_draft_list(request):
    posts = Post.objects.filter(published_date__isnull=True).order_by('-created_date')
    return render(request, 'blog/post_draft_list.html', {'posts': posts})
```

第四步添加一个template，新建blog/templates/blog/post_draft_list.html，内容如下：

```html
@% extends 'blog/base.html' %@
@% block content %@
    @% for post in posts %@
        <div class="post">
            <p class="date">created: @@ post.created_date|date:'d-m-Y' @@</p>
            <h1><a href="@% url 'blog.views.post_detail' pk=post.pk %@">@@ post.title @@</a></h1>
            <p>@@ post.text|truncatechars:200 @@</p>
        </div>
    @% endfor %@
@% endblock %@
```

这个模板跟我们的post_list.html非常相似。

刷新首页，点击那个草稿箱链接，看看效果。

![](https://xnstatic-1253397658.file.myqcloud.com/dj024.jpg)

## 发布功能

在文章详情页面添加一个发布的按钮，如果觉得合适的时候就能发布文章了。
每个新功能都是四部曲，你照着这四步做就行，你会发现越来越简单。

第一步在页面上添加一个链接或Form表单，这里我们添加一个链接。

打开blog/template/blog/post_detail.html，将下面这段

```
@% if post.published_date %@
    @@ post.published_date @@
@% endif %@
```

换成下面这段：

```html
@% if post.published_date %@
@@ post.published_date @@
@% else %@
<a class="btn btn-default" href="@% url 'blog.views.post_publish' pk=post.pk %@">Publish</a>
@% endif %@
```

这里增加了一个else语句，意思是如果没有发布日期的话就增加一个发布按钮。

第二步添加urls配置，打开blog/urls.py：

```python
url(r'^post/(?P<pk>[0-9]+)/publish/$', views.post_publish, name='post_publish'),
```

第三步视图，打开blog/views.py：

```python
def post_publish(request, pk):
    post = get_object_or_404(Post, pk=pk)
    post.publish()
    return redirect('blog.views.post_detail', pk=pk)
```

第四步模板，由于这次没有引入新的模板，所以这步省略。

刷新后看效果：

![](https://xnstatic-1253397658.file.myqcloud.com/dj025.jpg)

发布之后的效果：

![](https://xnstatic-1253397658.file.myqcloud.com/dj026.jpg)

注意观察发布前和发布后文章的发布日期那个位置的变化。并且发布后再去首页看看，文章已经可以正常显示了。

## 删除功能

最后当然需要一个删除功能了，还是四部曲！

第一步是在页面上添加链接，打开blog/templates/blog/post_detail.html，在编辑按钮下面一行添加如下：

```html
<a class="btn btn-default" href="@% url 'post_remove' pk=post.pk %@">
<span class="glyphicon glyphicon-remove"></span></a>
```

第二步配置urls映射，打开blog/urls.py，添加如下一行：

```python
url(r'^post/(?P<pk>[0-9]+)/remove/$', views.post_remove, name='post_remove'),
```

第三步添加视图view，打开blog/views.py，添加一个视图函数：

```python
def post_remove(request, pk):
    post = get_object_or_404(Post, pk=pk)
    post.delete()
    return redirect('blog.views.post_list')
```

第四步模板，由于这次又没有新的模板，所有这步省略。

OK，刷新页面看效果：

![](https://xnstatic-1253397658.file.myqcloud.com/dj027.jpg)

删除后再去首页看，已经没有这篇文章了。

## 分页功能

在首页显示文章列表时候需要分页显示，这时候可以使用django内置的Paginator来分页

关于分页的官方文档：<https://docs.djangoproject.com/en/1.9/topics/pagination/>

设置非常简单，简直是简单到变态。

### 在view里面使用Paginator

```python
from django.core.paginator import Paginator, EmptyPage, PageNotAnInteger

def post_list(request):
    """所有已发布文章"""
    postsAll = Post.objects.annotate(num_comment=Count('comment')).filter(
        published_date__isnull=False).prefetch_related(
        'category').prefetch_related('tags').order_by('-published_date')
    for p in postsAll:
        p.click = cache_manager.get_click(p)
    paginator = Paginator(postsAll, 10)  # Show 10 contacts per page
    page = request.GET.get('page')
    try:
        posts = paginator.page(page)
    except PageNotAnInteger:
        # If page is not an integer, deliver first page.
        posts = paginator.page(1)
    except EmptyPage:
        # If page is out of range (e.g. 9999), deliver last page of results.
        posts = paginator.page(paginator.num_pages)
    return render(request, 'blog/post_list.html', {'posts': posts, 'page': True})
```

这里我传到页面去的posts是一个Page对象，另外我还传了一个"page"标志，因为其他方法也会使用到这个页面，但是不需要分页的。

修改post_list.html页面，增加分页div

```html
@% for post in posts %@
...这个中间是对于文章post的循环，这个不变...
@% endfor %@
@% if page %@
    <div class="pagination">
        <span class="step-links">
            @% if posts.has_previous %@
                <a href="?page=@@ posts.previous_page_number @@">previous</a>
            @% endif %@
            <span class="current">
                Page @@ posts.number @@ of @@ posts.paginator.num_pages @@.
            </span>
            @% if posts.has_next %@
                <a href="?page=@@ posts.next_page_number @@">next</a>
            @% endif %@
        </span>
    </div>
    @% endif %@
```

刷新下列表首页，看看分页效果。

恭喜你，你可以大声对自己喊：我太棒了。^_^
