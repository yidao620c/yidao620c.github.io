---
title: WGSI简易教程
date: 2016-04-22 20:02:42 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

WSGI的全称是Web Server Gateway Interface，翻译过来就是Web服务器网关接口。具体的来说，WSGI是一个规范，
定义了Web服务器如何与Python应用程序进行交互，使得使用Python写的Web应用程序可以和Web服务器对接起来。
最新版本在[PEP-3333](https://www.python.org/dev/peps/pep-3333/)中定义。

对于初学者来说，上面那段就是废话，说了跟没说一样。接下来详细说明下这个东东到底是如何工作的。
<!-- more -->

### 为什么需要WSGI这个规范

在Web部署的方案上，有一个方案是目前应用最广泛的：

* 首先，部署一个Web服务器专门用来处理HTTP协议层面相关的事情，比如如何在一个物理机上提供多个不同的Web服务（单IP多域名，单IP多端口等）这种事情。
* 然后，部署一个用各种语言编写（Java, PHP, Python, Ruby等）的应用程序，这个应用程序会从Web服务器上接收客户端的请求，处理完成后，再返回响应给Web服务器，最后由Web服务器返回给客户端。

那么，要采用这种方案，Web服务器和应用程序之间就要知道如何进行交互。为了定义Web服务器和应用程序之间的交互过程，就形成了很多不同的规范。这种规范里最早的一个是CGI，1993年开发的。
后来又出现了很多这样的规范。比如改进CGI性能的FasgCGI，Java专用的Servlet规范，还有Python专用的WSGI规范等。提出这些规范的目的就是为了定义统一的标准，提升程序的可移植性。

### WSGI如何工作

从上文可以知道，WSGI相当于是Web服务器和Python应用程序之间的桥梁。那么这个桥梁是如何工作的呢？首先，我们明确桥梁的作用，WSGI存在的目的有两个：

1. 让Web服务器知道如何调用Python应用程序，并且把用户的请求告诉应用程序。
2. 让Python应用程序知道用户的具体请求是什么，以及如何返回结果给Web服务器。

#### WSGI中的角色

在WSGI中定义了两个角色，Web服务器端称为server或者gateway，应用程序端称为application或者framework（因为WSGI的应用程序端的规范一般都是由具体的框架来实现的）。我们下面统一使用server和application这两个术语。

server端会先收到用户的请求，然后会根据规范的要求调用application端，如下图所示：
![](https://xnstatic-1253397658.file.myqcloud.com/wsgi01.png)
调用的结果会被封装成HTTP响应后再发送给客户端。

#### server如何调用application

首先，每个application的入口只有一个，也就是所有的客户端请求都同一个入口进入到应用程序。

接下来，server端需要知道去哪里找application的入口。这个需要在server端指定一个Python模块，也就是Python应用中的一个文件，并且这个模块中需要包含一个名称为application的可调用对象（函数和类都可以），这个application对象就是这个应用程序的唯一入口了。WSGI还定义了application对象的形式：

```python
def simple_app(environ, start_response):
    pass
```

上面代码中的`environ`和`start_response`就是server端调用application对象时传递的两个参数。

我们来看具体的例子。假设我们的应用程序的入口文件是/var/www/index.py，那么我们就需要在server端配置好这个路径（如何配置取决于server端的实现），然后在index.py中的代码如下所示：

使用标准库，也就是python内置的一个wsgi参考实现，实现了标准的wsgi协议（这个只是demo）

```python
import wsgiref

application = wsgiref.simple_server.demo_app
```

使用django框架

```python
import os

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")

from django.core.wsgi import get_wsgi_application

application = get_wsgi_application()
```

不管哪种方式，文件中都需要有一个名字为`application`的可调用对象，server端会找到这个文件，然后调用这个对象。

#### application对象需要做什么

application对象需要是一个可调用对象，而且其定义需要满足如下形式：

```python
def simple_app(environ, start_response):
    pass
```

比如去查看django框架源码，就会发现`get_wsgi_application()`返回了一个`WSGIHandler`可调用对象，这个对象有下面的方法：

```python
def __call__(self, environ, start_response):
    # Set up middleware if needed. We couldn't do this earlier, because
    # settings weren't available.
    ...
    response = self.get_response(request)
    status = '%s %s' % (response.status_code, response.reason_phrase)
    response_headers = [(str(k), str(v)) for k, v in response.items()]
    for c in response.cookies.values():
        response_headers.append((str('Set-Cookie'), str(c.output(header=''))))
    start_response(force_str(status), response_headers)  # 这一步是返回HTTP状态码和header部分
    return response  # 这一步返回body
```

#### environ参数

environ参数是一个Python的字典，里面存放了所有和客户端相关的信息，这样application对象就能知道客户端请求的资源是什么，请求中带了什么数据等。

我们来看一些environ中常用的成员。

首先是CGI规范中要求的变量：

```
REQUEST_METHOD： 请求方法，是个字符串，'GET', 'POST'等
SCRIPT_NAME： HTTP请求的path中的用于查找到application对象的部分，比如Web服务器可以根据path的一部分来决定请求由哪个virtual host处理
PATH_INFO： HTTP请求的path中剩余的部分，也就是application要处理的部分
QUERY_STRING： HTTP请求中的查询字符串，URL中?后面的内容
CONTENT_TYPE： HTTP headers中的content-type内容
CONTENT_LENGTH： HTTP headers中的content-length内容
SERVER_NAME和SERVER_PORT： 服务器名和端口，这两个值和前面的SCRIPT_NAME, PATH_INFO拼起来可以得到完整的URL路径
SERVER_PROTOCOL： HTTP协议版本，HTTP/1.0或者HTTP/1.1
HTTP_： 和HTTP请求中的headers对应。
```

WSGI规范中还要求environ包含下列成员：

```
wsgi.version：表示WSGI版本，一个元组(1, 0)，表示版本1.0
wsgi.url_scheme：http或者https
wsgi.input：一个类文件的输入流，application可以通过这个获取HTTP request body
wsgi.errors：一个输出流，当应用程序出错时，可以将错误信息写入这里
wsgi.multithread：当application对象可能被多个线程同时调用时，这个值需要为True
wsgi.multiprocess：当application对象可能被多个进程同时调用时，这个值需要为True
wsgi.run_once：当server期望application对象在进程的生命周期内只被调用一次时，该值为True
```

上面列出的这些内容已经包括了客户端请求的所有数据，足够application对象处理客户端请求了。

#### start_resposne参数

start_response是一个可调用对象，接收两个必选参数和一个可选参数：

```
status: 一个字符串，表示HTTP响应状态字符串
response_headers: 一个列表，包含有如下形式的元组：(header_name, header_value)，用来表示HTTP响应的headers
exc_info（可选）: 用于出错时，server需要返回给浏览器的信息
```

在application对象将body作为返回值return之前，需要先调用start_response()
，将status和headers的内容返回给server，这同时也是告诉server，application对象要开始返回body了。

#### application对象的返回值

application对象的返回值用于为HTTP响应提供body，如果没有body，那么可以返回None。如果有body的化，那么需要返回一个可迭代的对象。server端通过遍历这个可迭代对象可以获得body的全部内容。

上面我分析的django源码的注释可以看到

```python
start_response(force_str(status), response_headers)
```

这一步发送HTTP响应的Header，注意Header只能发送一次，也就是只能调用一次start_response()函数。start_response()
函数接收两个参数，一个是HTTP响应码，一个是一组list表示的HTTP Header，每个Header用一个包含两个str的tuple表示。
通常情况下，都应该把Content-Type头发送给浏览器。其他很多常用的HTTP Header也应该发送。

最后面一句返回body，也可以为空。

```python
return response
```

### WSGI中间件

WSGI Middleware（中间件）也是WSGI规范的一部分。我们之前讲过WSGI中的两个角色：server和application。
而middleware是运行在server和application中间的应用（一般也是python应用）。middleware同时具备server和application角色，
对于server来讲它就是个application，而对于application来说，它就是个server。middleware并不修改server端和application端的规范，
只是同时实现了这两种角色的功能而已。你可以将middleware形象比喻成批发商，对于厂商而已它是购买者，而对于零售店而已它是供应方。
下面我用一张图来形象的说明这个中间件的工作原理：
![](https://xnstatic-1253397658.file.myqcloud.com/wsgi20.png)

上图中最上面的三个彩色框表示角色，中间的白色框表示操作，操作的发生顺序按照1 ~ 5进行了排序，我们直接对着上图来说明middleware是如何工作的：

1. Server收到客户端的HTTP请求后，生成了environ_s，并且已经定义了start_response_s。
2. Server调用Middleware的application对象，传递的参数是environ_s和start_response_s。
3. Middleware会根据environ执行业务逻辑，生成environ_m，并且已经定义了start_response_m。
4.

Middleware决定调用Application的application对象，传递参数是environ_m和start_response_m。Application的application对象处理完成后，会调用start_response_m并且返回结果给Middleware，存放在result_m中。

5. Middleware处理result_m，然后生成result_s，接着调用start_response_s，并返回结果result_s给Server端。Server端获取到result_s后就可以发送结果给客户端了。

从上面的流程可以看出middleware应用的几个特点：

1. Server认为middleware是一个application。
1. Application认为middleware是一个server。
1. Middleware可以有多层。

因为Middleware能过处理所有经过的request和response，所以要做什么都可以，没有限制。比如可以检查request是否有非法内容，检查response是否有非法内容，为request加上特定的HTTP
header等，这些都是可以的。

### WSGI的实现和部署

要使用WSGI，需要分别实现server角色和application角色。

Application端的实现一般是由Python的各种框架来实现的，比如Django,
web.py等，一般开发者不需要关心WSGI的实现，框架会会提供接口让开发者获取HTTP请求的内容以及发送HTTP响应。

Server端的实现会比较复杂一点，这个主要是因为软件架构的原因。一般常用的Web服务器，如Apache和nginx，都不会内置WSGI的支持，而是通过扩展来完成。比如Apache服务器，会通过扩展模块mod_wsgi来支持WSGI。
Apache和mod_wsgi之间通过程序内部接口传递信息，mod_wsgi会实现WSGI的server端、进程管理以及对application的调用。Nginx上一般是用proxy的方式，用nginx的协议将请求封装好，发送给应用服务器，比如uWSGI，应用服务器会实现WSGI的服务端、进程管理以及对application的调用。

### WSGI小例子

上面讲了这么多废话，不然来个实际例子看看。

Python内置了一个WSGI服务器，这个模块叫wsgiref，它是用纯Python编写的WSGI服务器的参考实现。所谓"参考实现"
是指该实现完全符合WSGI标准，但是不考虑任何运行效率，仅供开发和测试使用。

我们先编写hello.py，实现Web应用程序的WSGI处理函数：

```python
# hello.py

def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    # 这里我们使用PATH_INFO变量获取请求的URL
    body = '<h1>Hello, %s!</h1>' % (environ['PATH_INFO'][1:] or 'web')
    return [body.encode('utf-8')]
```

然后，再编写一个server.py，负责启动WSGI服务器，加载application()函数：

```python
# server.py
# 从wsgiref模块导入:
from wsgiref.simple_server import make_server
# 导入我们自己编写的application函数:
from hello import application

# 创建一个服务器，IP地址为空，端口是8000，处理函数是application:
httpd = make_server('', 8000, application)
print('Serving HTTP on port 8000...')
# 开始监听HTTP请求:
httpd.serve_forever()
```

确保以上两个文件在同一个目录下，然后在命令行输入python server.py来启动WSGI服务器：
![](https://xnstatic-1253397658.file.myqcloud.com/wsgi02.png)
启动成功后，打开浏览器，输入`http://localhost:8000/xiongneng` 就可以看到结果了：
![](https://xnstatic-1253397658.file.myqcloud.com/wsgi03.png)

是不是有点Web App的感觉了？其实WSGI就是这么简单！

参考：

* [WSGI简介](https://segmentfault.com/a/1190000003069785)
* [WSGI接口](http://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001432012393132788f71e0edad4676a3f76ac7776f3a16000)

