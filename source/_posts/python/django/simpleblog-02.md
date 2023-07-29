---
title: Django1.9开发博客02- 模型
date: 2015-08-03 18:49:57 +0800
toc: true
categories: [ python ]
tags: [ django ]
---

django的模型就是用于在数据库中存储的某种类型的对象。在我们的博客系统中，
发表的文章就是一个模型，需要存储在数据库中。
这里我们使用django默认的sqlite3库，对于我们的这个小系统而言已经足够了。
<!-- more -->

### 创建一个应用

在django中有两个概念需要弄清楚。一个是工程（project）的概念，一个是应用（application）的概念。
它们的关系是：一个工程中包含多个应用。每个应用都是独立的，应用通过setting.py注册到工程中来就可以使用了。
这样可以解耦合，并且好的应用也可以复用。很好的模块化设计！

在manage.py文件所在目录，执行下面命令：

```
(myvenv) [mango@centos00 mysite]$ python manage.py startapp blog
```

你会看到一个新的blog文件夹被创建，并且下面多了许多文件，目前结构如下：

```
mysite
├── mysite
|       __init__.py
|       settings.py
|       urls.py
|       wsgi.py
├── manage.py
└── blog
    ├── migrations
    |       __init__.py
    ├── __init__.py
    ├── admin.py
    ├── models.py
    ├── tests.py
    └── views.py
```

然后在setting.py中注册这个应用

```python
INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'blog',
)
```

### 创建一个博客文章的模型

在blog/models.py中定义所有的模型，用vim打开后添加下面的内容

```python
from django.db import models
from django.utils import timezone
from django.contrib.auth.models import User

class Post(models.Model):
    author = models.ForeignKey(User)
    title = models.CharField(max_length=200)
    text = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)
    published_date = models.DateTimeField(blank=True, null=True)

    def publish(self):
        self.published_date = timezone.now()
        self.save()

    def __str__(self):
        return self.title
```

我们定义了作者，标题，文章内容，创建时间，发布时间等。还添加了一个publish方法用于保存发布。
每个字段都需要定义它的类型，这里的几个类型解释如下：

```
CharField：普通的文本
TextField：长文本
DateTimeField：日期时间类型
ForeignKey：外键类型
```

详细的字段类型说明请参考官方文档：[field-types](https://docs.djangoproject.com/en/1.9/ref/models/fields/#field-types)

**在数据库中为模型生成表结构：**

每次我们新建了一个模型后，需要在数据库中添加对应的表。

第一步是先让django感知到我们刚刚已经创建了一个新的模型：

```bash
(myvenv) [mango@centos00 mysite]$ python manage.py makemigrations blog
```

输出如下：

```
Migrations for 'blog':
  0001_initial.py:
    - Create model Post
```

这时候django已经为我们准备好了数据库更新的sql文件。

第二步是让django帮我们执行这些文件：

```bash
(myvenv) [mango@centos00 mysite]$ python manage.py migrate blog
```

输出如下：

```
Operations to perform:
  Apply all migrations: blog
Running migrations:
  Applying blog.0001_initial... OK
```

OK，这时候数据库中已经有post这张表了。

### 对象关系映射ORM

下面我们看看django的ORM是怎样和数据库打交道的。

首先解释下QuerySet，它是某个模型的对象列表，用来从数据库中读取数据，过滤和排序等等。

### Django控制台Django Shell

执行以下命令可以打开django的控制台

```bash
(myvenv) [mango@centos00 mysite]$ python manage.py shell
```

查询所有的博客文章

```python
>> > from blog.models import Post
>> > Post.objects.all()
[]
```

这时候肯定是空的，因为我们还没有任何数据。

下面我们去新建几篇文章

先要创建一个用户，因为Post里面有User外键

```python
>> > from django.contrib.auth.models import User
>> > User.objects.create(username='ola')
< User: ola >
>> > User.objects.all()
[ < User: ola >]
```

然后添加文章：

```python
>> > user = User.objects.get(username='ola')
>> > Post.objects.create(author=user, title='Sample title', text='Test')
< Post: Sample
title >
>> > Post.objects.all()
[ < Post: Sample
title >]
```

**文章过滤**

先像前面那样再添加几篇文章，此处省略七十二个字….

```
>>> Post.objects.filter(author=user)
[<Post: Sample title>, <Post: Cool day Aha>, <Post: LiLei and Hanmeimei>, <Post: Happy boy>]
>>> Post.objects.filter(title__contains='LiLei')
[<Post: LiLei and Hanmeimei>]
>>> Post.objects.filter(published_date__isnull=False)
[]
```

最后的输出表示所有的文章published_date都是空的。我想改变这个情况。可以这样做

```
>>> post = Post.objects.get(id=1)
>>> post.publish()
>>> Post.objects.filter(published_date__isnull=False)
[<Post: Sample title>]
```

**结果排序**

还可以根据某个或某几个字段来排序检索结果

```
>>> Post.objects.order_by('created_date')
[<Post: Sample title>, <Post: Cool day Aha>, <Post: LiLei and Hanmeimei>, <Post: Happy boy>]
>>> Post.objects.order_by('-created_date')
[<Post: Happy boy>, <Post: LiLei and Hanmeimei>, <Post: Cool day Aha>, <Post: Sample title>]
```

OK，目前为止你应该对django的ORM有了大体的了解了。

请用exit()退出django的控制台

```
>>> exit()
```

### 利用django admin修改模型

在上面我们已经创建了Post模型并且通过django控制台来添加修改模型。
然后我们使用django自带的web管理界面admin来在页面上修改模型数据。

### 模型注册

首先我们需要在admin中注册对应的模型，打开blog/admin.py文件，修改如下

```python
from django.contrib import admin
from .models import Post

admin.site.register(Post)
```

注意上面的from .models import Post，使用到了python3的相对导入。

### 运行服务器

一切都这么简单，接下来我们启动服务器

```bash
(myvenv) [mango@centos mysite]$ python manage.py runserver 192.168.203.95:8000
```

打开链接：http://192.168.203.95:8000/admin/

看到下面的界面：

![](https://xnstatic-1253397658.file.myqcloud.com/dj002.jpg)

### 添加管理员

不过你需要一个管理员才能登录。运行python manage.py createsuperuser可以创建管理员账号。

```bash
(myvenv) [mango@centos00 mysite]$ python manage.py createsuperuser
Username (leave blank to use 'mango'): admin
Email address: admin@gmail.com
Password:
Password (again):
Superuser created successfully.
```

我创建了一个admin/admin的账户。这时候登录

![](https://xnstatic-1253397658.file.myqcloud.com/dj003.jpg)

点击Posts修改或者增加等等，确保里面至少2个又published_date，这个后面会用到。

![](https://xnstatic-1253397658.file.myqcloud.com/dj004.jpg)

更多关于django admin的内容，参考官方文档：[admin](https://docs.djangoproject.com/en/1.9/ref/contrib/admin/)

### 接下来呢？

先别急嘛，坐下来喝杯咖啡先。到现在为止你已经前进了很远了。

