---
title: python核心 - web开发
date: 2015-12-30 22:22:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

Web应用开发可以说是目前软件开发中最重要的部分。Web开发也经历了好几个阶段：静态Web页面、CGI、ASP/JSP/PHP、MVC。

目前，Web开发技术仍在快速发展中，异步开发、新的MVVM前端技术层出不穷。

Python的诞生历史比Web还要早，由于Python是一种解释型的脚本语言，开发效率高，所以非常适合用来做Web开发。

Python有上百种Web开发框架，有很多成熟的模板技术，选择Python开发Web应用，不但开发效率高，而且运行速度快。
<!-- more -->

## WSGI接口

一个Web应用的本质就是

1. 浏览器发送一个HTTP请求；
2. 服务器收到请求，生成一个HTML文档；
3. 服务器把HTML文档作为HTTP响应的Body发送给浏览器；
4. 浏览器收到HTTP响应，从HTTP Body取出HTML文档并显示。

所以，最简单的Web应用就是先把HTML用文件保存好，用一个现成的HTTP服务器软件，接收用户请求，从文件中读取HTML返回。
Apache、Nginx、Lighttpd等这些常见的静态服务器就是干这件事情的。

如果要动态生成HTML，就需要把上述步骤自己来实现。不过，接受HTTP请求、解析HTTP请求、发送HTTP响应都是苦力活，
如果我们自己来写这些底层代码，还没开始写动态HTML呢，就得花个把月去读HTTP规范。

正确的做法是底层代码由专门的服务器软件实现，我们用Python专注于生成HTML文档。
因为我们不希望接触到TCP连接、HTTP原始请求和响应格式，所以，需要一个统一的接口，让我们专心用Python编写Web业务。

这个接口就是WSGI：Web Server Gateway Interface。

关于WSGI我有专门的文章讲解，这里就不重复了，感兴趣的可以去翻翻。

## Web框架

其实一个Web App，就是写一个WSGI的处理函数，针对每个HTTP请求进行响应。
但是直接自己编写WSGI处理函数会显得非常繁琐，这里完全可以用框架代替我们做。

由于用Python开发一个Web框架十分容易，所以Python有上百个开源的Web框架。
这里我们先不讨论各种Web框架的优缺点，直接选择一个比较流行的Web框架——Flask来使用。

用Flask编写Web App比WSGI接口简单（这不是废话么，要是比WSGI还复杂，用框架干嘛？），我们先用pip安装Flask：

```bash
pip install flask
```

然后写一个app.py，处理3个URL，分别是：

* GET /：首页，返回Home；
* GET /signin：登录页，显示登录表单；
* POST /signin：处理登录表单，显示登录结果。

注意上面，同一个URL/signin分别有GET和POST两种请求，映射到两个处理函数中。

Flask通过Python的装饰器在内部自动地把URL和函数给关联起来，所以，我们写出来的代码就像这样：

```python
from flask import Flask
from flask import request

app = Flask(__name__)


@app.route('/', methods=['GET', 'POST'])
def home():
    return '<h1>Home</h1>'


@app.route('/signin', methods=['GET'])
def signin_form():
    return '''<form action="/signin" method="post">
              <p><input name="username"></p>
              <p><input name="password" type="password"></p>
              <p><button type="submit">Sign In</button></p>
              </form>'''


@app.route('/signin', methods=['POST'])
def signin():
    # 需要从request对象读取表单内容：
    if request.form['username'] == 'admin' and request.form['password'] == 'password':
        return '<h3>Hello, admin!</h3>'
    return '<h3>Bad username or password.</h3>'


if __name__ == '__main__':
    app.run()
```

运行`python app.py`，Flask自带的Server在端口5000上监听，
打开浏览器，输入首页地址 <http://localhost:5000/> 看看效果。

再在浏览器地址栏输入<http://localhost:5000/signin>，会显示登录表单。
输入预设的用户名admin和口令password，登录成功，输入其他就显示登陆失败。

实际的Web App应该拿到用户名和口令后，去数据库查询再比对，来判断用户是否能登录成功。

除了Flask，常见的Python Web框架还有：

* [Django](https://www.djangoproject.com/)：全能型Web框架；
* [web.py](http://webpy.org/)：一个小巧的Web框架；
* [Bottle](http://bottlepy.org/)：和Flask类似的Web框架；
* [Tornado](http://www.tornadoweb.org/)：Facebook的开源异步Web框架。

有了Web框架，我们在编写Web应用时，注意力就从WSGI处理函数转移到URL+对应的处理函数，这样，编写Web App就更加简单了。

## 使用模板

在函数中返回一个包含HTML的字符串，简单的页面还可以，
但是，一些复杂的web应用成百上千个HTML页面，你确信能在Python的字符串中正确地写出来么？反正我是做不到。

有Web开发经验的同学都明白，Web App最复杂的部分就在HTML页面。HTML不仅要正确，还要通过CSS美化，
再加上复杂的JavaScript脚本来实现各种交互和动画效果。总之，生成HTML页面的难度很大。

由于在Python代码里拼字符串是不现实的，所以，模板技术出现了。

使用模板，我们需要预先准备一个HTML文档，这个HTML文档不是普通的HTML，
而是嵌入了一些变量和指令，然后，根据我们传入的数据，替换后，得到最终的HTML，发送给用户：
![](https://xnstatic-1253397658.file.myqcloud.com/pyweb001.png)
这就是传说中的MVC：Model-View-Controller，中文名"模型-视图-控制器"。

Python处理URL的函数就是C：Controller，Controller负责业务逻辑，比如检查用户名是否存在，取出用户信息等等；

包含变量@@ name @@的模板就是V：View，View负责显示逻辑，通过简单地替换一些变量，View最终输出的就是用户看到的HTML。

MVC中的Model在哪？Model是用来传给View的，这样View在替换变量的时候，就可以从Model中取出相应的数据。

上面的例子中，Model就是一个dict：`{ 'name': 'Michael' }`

现在，我们把上次直接输出字符串作为HTML的例子用高端大气上档次的MVC模式改写一下：

```python
from flask import Flask, request, render_template

app = Flask(__name__)


@app.route('/', methods=['GET', 'POST'])
def home():
    return render_template('home.html')


@app.route('/signin', methods=['GET'])
def signin_form():
    return render_template('form.html')


@app.route('/signin', methods=['POST'])
def signin():
    username = request.form['username']
    password = request.form['password']
    if username == 'admin' and password == 'password':
        return render_template('signin-ok.html', username=username)
    return render_template('form.html', message='Bad username or password', username=username)


if __name__ == '__main__':
    app.run()
```

Flask通过render_template()函数来实现模板的渲染。和Web框架类似，
Python的模板也有很多种。Flask默认支持的模板是jinja2，所以我们先直接安装jinja2：

```bash
pip install jinja2
```

然后，开始编写jinja2首页模板：

```html home.html

<html>
<head>
    <title>Home</title>
</head>
<body>
<h1 style="font-style:italic">Home</h1>
</body>
</html>
```

用来显示登录表单的模板：

```html form.html

<html>
<head>
    <title>Please Sign In</title>
</head>
<body>
@% if message %@
<p style="color:red">@@ message @@</p>
@% endif %@
<form action="/signin" method="post">
    <legend>Please sign in:</legend>
    <p><input name="username" placeholder="Username" value="@@ username @@"></p>
    <p><input name="password" placeholder="Password" type="password"></p>
    <p>
        <button type="submit">Sign In</button>
    </p>
</form>
</body>
</html>
```

登录成功的模板：

```html signin-ok.html

<html>
<head>
    <title>Welcome, @@ username @@</title>
</head>
<body>
<p>Welcome, @@ username @@!</p>
</body>
</html>
```

登录失败的模板呢？我们在form.html中加了一点条件判断，把form.html重用为登录失败的模板。

最后，一定要把模板放到正确的templates目录下，templates和app.py在同级目录下：
![](https://xnstatic-1253397658.file.myqcloud.com/pyweb002.png)

启动python app.py，看看使用模板的页面效果，不重复演示了

有了MVC，我们就分离了Python代码和HTML代码。HTML代码全部放到模板里，写起来更有效率。

