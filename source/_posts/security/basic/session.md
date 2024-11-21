---
title: 【安全贴士】聊聊Web会话管理
date: 2017-06-20 09:55:19 +0800
toc: true
categories: [ 网络安全 ]
tags: [ 安全贴士 ]
---

Web刚刚兴起的时候，服务器只提供一些简单的HTML页面和链接，用户打开网址去浏览。
并不需要记住每次请求是谁发送来的，每次请求对服务器来讲都是全新的。
既然是浏览，作为一个服务器，为什么要记住谁在一段时间里都浏览了什么文档呢？

但是好日子没持续多久，很快大家就不满足于静态的HTML文档了，交互式的Web应用开始兴起，尤其是论坛，在线购物等网站。
必须管理会话，必须记住哪些人登录系统，哪些人往自己的购物车中放了商品，也就是说我必须把每个人区分开。

由于HTTP协议的无状态特性，必须加点小手段，才能完成会话管理。

本文总结了3种常见的实现web应用会话管理的方式：

1. 基于server端session的管理方式
2. 基于cookie的管理方式
3. 基于token的管理方式

目的是加深对web中用户登录机制的理解，对实际项目开发也有参考价值。
<!-- more -->

## session和cookie

在讲解着三种方式之前，先科普一下几个知识。

1. 由于HTTP协议是无状态的协议，所以服务端需要记录用户的状态时，就需要用某种机制来识具体的用户，这个机制就是Session。
   Session是保存在服务端的，有一个唯一标识。在大型的网站，一般会有专门的Session服务器集群，这个时候 Session 信息都是放在内存的。

2. 思考一下服务端如何识别特定的客户？这个时候Cookie就登场了。每次HTTP请求的时候，客户端都会发送相应的Cookie信息到服务端。
   实际上大多数的应用都是用 Cookie 来实现Session跟踪的，第一次创建Session的时候，服务端会在HTTP协议中告诉客户端，
   需要在 Cookie 里面记录一个Session ID，以后每次请求把这个会话ID发送到服务器，我就知道你是谁了。
   有人问，如果客户端的浏览器禁用了 Cookie 怎么办？一般这种情况下，会使用一种叫做URL重写的技术来进行会话跟踪，
   即每次HTTP交互，URL后面都会被附加上一个诸如 sid=xxxxx 这样的参数，服务端据此来识别用户。

3. Cookie其实还可以用在一些方便用户的场景下，设想你某次登陆过一个网站，下次登录的时候不想再次输入账号了，怎么办？
   这个信息可以写到Cookie里面，访问网站的时候，网站页面的脚本可以读取这个信息，就自动帮你把用户名给填了，能够方便一下用户。
   这也是Cookie名称的由来，给用户的一点甜头。

所以，总结一下：

1. session 在服务器端，cookie 在客户端（浏览器）
2. session 默认被存在在服务器的一个文件里（不是内存）
3. session 的运行依赖 session id，而 session id 是存在cookie中的，也就是说，如果浏览器禁用了cookie，
   同时 session 也会失效（但是可以通过其它方式实现，比如在 url 中传递 session_id）
4. session 可以放在 文件、数据库、或内存中都可以。
5. 用户验证这种场合一般会用 session 因此，维持一个会话的核心就是客户端的唯一标识，即 session id

## 基于server端session

在早期web应用中，通常使用服务端session来管理用户的会话。

1）服务端session是用户第一次访问应用时，服务器就会创建的对象，代表用户的一次会话过程，可以用来存放数据。
服务器为每一个session都分配一个唯一的sessionid，以保证每个用户都有一个不同的session对象。

2）服务器在创建完session后，会把sessionid通过cookie返回给用户所在的浏览器，这样当用户第二次及以后向服务器发送请求的时候，
就会通过cookie把sessionid传回给服务器，以便服务器能够根据sessionid找到与该用户对应的session对象。

3）session通常有失效时间的设定，比如2个小时。当失效时间到，服务器会销毁之前的session，并创建新的session返回给用户。
但是只要用户在失效时间内，有发送新的请求给服务器，通常服务器都会把他对应的session的失效时间根据当前的请求时间再延长2个小时。

4）session在一开始并不具备会话管理的作用。它只有在用户登录认证成功之后，并且往sesssion对象里面放入了用户登录成功的凭证，
才能用来管理会话。管理会话的逻辑也很简单，只要拿到用户的session对象，看它里面有没有登录成功的凭证，就能判断这个用户是否已经登录。
当用户主动退出的时候，会把它的session对象里的登录凭证清掉。
所以在用户登录前或退出后或者session对象失效时，肯定都是拿不到需要的登录凭证的。

可简单使用流程图描述如下：

![](https://xnstatic-1253397658.file.myqcloud.com/session01.png)

主流的web开发平台都原生支持这种会话管理的方式，而且开发起来很简单。它还有一个比较大的优点就是安全性好，
因为在浏览器端与服务器端保持会话状态的媒介始终只是一个sessionid串，只要这个串够随机，攻击者就不能轻易冒充他人的sessionid进行操作；
除非通过CSRF或http劫持的方式，才有可能冒充别人进行操作；即使冒充成功，也必须被冒充的用户session里面包含有效的登录凭证才行。

但是这种方式也有几个问题需要解决：

1）这种方式将会话信息存储在web服务器里面，所以在用户同时在线量比较多时，这些会话信息会占据比较多的内存；

2）当应用采用集群部署的时候，会遇到多台web服务器之间如何做session共享的问题。因为session是由单个服务器创建的，
但是处理用户请求的服务器不一定是那个创建session的服务器，这样他就拿不到之前已经放入到session中的登录凭证之类的信息了；

3）多个应用要共享session时，除了以上问题，还会遇到跨域问题，因为不同的应用可能部署的主机不一样，需要在各个应用做好cookie跨域的处理。

针对问题1和问题2，我见过的解决方案是采用redis/memcached这种中间服务器来管理session的增删改查，
一来减轻web服务器的负担，二来解决不同web服务器共享session的问题。

针对问题3，由于服务端的session依赖cookie来传递sessionid，所以在实际项目中，只要解决各个项目里面如何实现sessionid的cookie跨域访问即可，
这个是可以实现的，就是比较麻烦，前后端有可能都要做处理。

如果在一些小型的web应用中使用，可以不用考虑上面三个问题，所以很适合这种方式。

## 基于cookie

由于前一种方式会增加服务器的负担和架构的复杂性，所以后来就有人想出直接把用户的登录凭证直接存到客户端的方案，
当用户登录成功之后，把登录凭证写到cookie里面，并给cookie设置有效期，后续请求直接验证存有登录凭证的cookie是否存在以及凭证是否有效，
即可判断用户的登录状态。使用它来实现会话管理的整体流程如下：

1）用户发起登录请求，服务端根据传入的用户密码之类的身份信息，验证用户是否满足登录条件，如果满足，就根据用户信息创建一个登录凭证，
这个登录凭证简单来说就是一个对象，最简单的形式可以只包含用户id，凭证创建时间和过期时间三个值。

2）服务端把上一步创建好的登录凭证，先对它做数字签名，然后再用对称加密算法做加密处理，将签名、加密后的字串，写入cookie。
cookie的名字必须固定（如ticket），因为后面再获取的时候，还得根据这个名字来获取cookie值。
这一步添加数字签名的目的是防止登录凭证里的信息被篡改，因为一旦信息被篡改，那么下一步做签名验证的时候肯定会失败。
做加密的目的，是防止cookie被别人截取的时候，无法轻易读到其中的用户信息。

3）用户登录后发起后续请求，服务端根据上一步存登录凭证的cookie名字，获取到相关的cookie值。然后先做解密处理，再做数字签名的认证，
如果这两步都失败，说明这个登录凭证非法；如果这两步成功，接着就可以拿到原始存入的登录凭证了。然后用这个凭证的过期时间和当前时间做对比，
判断凭证是否过期，如果过期，就需要用户再重新登录；如果未过期，则允许请求继续。

![](https://xnstatic-1253397658.file.myqcloud.com/session02.png)

它的缺点也比较明显：

1）cookie有大小限制，存储不了太多数据

2）每次传送cookie，增加了请求的数量，对访问性能也有影响；

3）也有跨域问题，毕竟还是要用cookie。

相比起第一种方式，基于cookie方案明显还是要好一些，目前好多web开发平台或框架都默认使用这种方式来做会话管理。

跨域的问题可以用CORS（跨域资源共享）的方式来快速解决。

## 基于token

前面两种会话管理方式因为都用到cookie，不适合用在移动端native app里面，native app不好管理cookie，毕竟它不是浏览器。
这两种方案都不适合用来做纯api服务的登录认证，就要考虑第三种会话管理方式，也就是token认证。

这种方式从流程和实现上来说，跟cookie-based的方式没有太多区别，只不过cookie-based里面写到cookie里面的ticket在这种方式下称为token，
这个token在返回给客户端之后，后续请求都必须通过url参数或者是http header的形式，主动带上token，
这样服务端接收到请求之后就能直接从http header或者url里面取到token进行验证：

这种方式不通过cookie进行token的传递，而是每次请求的时候，主动把token加到http header里面或者url后面，
所以即使在native app里面也能使用它来调用我们通过web发布的api接口。app里面还要做两件事情：

1）有效存储token，得保证每次调接口的时候都能从同一个位置拿到同一个token；

2）每次调接口的的代码里都得把token加到header或者接口地址里面。

这种方式同样适用于网页应用，token可以存于localStorage或者sessionStorage里面，然后每发ajax请求的时候，
都把token拿出来放到ajax请求的header里即可。不过如果是非接口的请求，比如直接通过点击链接请求一个页面这种，
是无法自动带上token的。所以这种方式也仅限于走纯接口的web应用。

![](https://xnstatic-1253397658.file.myqcloud.com/session03.png)

## JWT

现在SPA应用，前后端完全分离，基于API接口的应用越来越多，这时候基于token的认证就是最好的选择方式了。
好在这个方式的技术其实早就有很多实现了，而且还有现成的标准可用，这个标准就是JWT(json-web-token)。

JWT本身并没有做任何技术实现，它只是定义了token-based的管理方式该如何实现，
它规定了token的应该包含的标准内容以及token的生成过程和方法。目前实现了这个标准的技术已经有非常多：

官方网站：<https://jwt.io/#libraries-io>

Java实现：<https://github.com/auth0/java-jwt>

