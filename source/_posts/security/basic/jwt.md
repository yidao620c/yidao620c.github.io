---
title: 【安全贴士】JWT介绍和安全防范
date: 2017-06-22 19:12:35 +0800
toc: true
categories: [ 网络安全 ]
tags: [ 安全贴士 ]
---

Json web token (JWT), 是为了在网络应用环境间传递声明而执行的一种基于JSON的开放标准（(RFC 7519).该token被设计为紧凑且安全的，
特别适用于分布式站点的单点登录（SSO）场景。JWT的声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，
以便于从资源服务器获取资源，也可以增加一些额外的其它业务逻辑所必须的声明信息，该token也可直接被用于认证，也可被加密。

JWT官网：<https://jwt.io/>
<!-- more -->

JWT是由三段信息构成的，将这三段信息文本用.链接一起就构成了Jwt字符串。就像这样:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

## JWT的构成

第一部分我们称它为头部（header),第二部分我们称其为载荷（payload, 类似于飞机上承载的物品)，第三部分是签证（signature)。

*header*

jwt的头部承载两部分信息：

* 声明类型，这里是jwt
* 声明加密的算法 通常直接使用 HMAC SHA256

这里的加密算法是单向函数散列算法，常见的有MD5、SHA、HAMC。这里使用基于密钥的Hash算法HMAC生成散列值。

* MD5 message-digest algorithm 5 （信息-摘要算法）缩写，广泛用于加密和解密技术，常用于文件校验。校验？不管文件多大，经过MD5后都能生成唯一的MD5值
* SHA (Secure Hash Algorithm，安全散列算法），数字签名等密码学应用中重要的工具，安全性高于MD5
* HMAC (Hash Message Authentication Code，散列消息鉴别码，基于密钥的Hash算法的认证协议。用公开函数和密钥产生一个固定长度的值作为认证标识，用这个标识鉴别消息的完整性。常用于接口签名验证

完整的头部就像下面这样的JSON：

```json
{
  'typ': 'JWT',
  'alg': 'HS256'
}
```

然后将头部进行base64加密（该加密是可以对称解密的),构成了第一部分

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9
```

*playload*

载荷就是存放有效信息的地方，这些有效信息包含三个部分：

* 标准中注册的声明
* 公共的声明
* 私有的声明

标准中注册的声明 (建议但不强制使用) ：

* iss: jwt签发者
* sub: jwt所面向的用户
* aud: 接收jwt的一方
* exp: jwt的过期时间，这个过期时间必须要大于签发时间
* nbf: 定义在什么时间之前，该jwt都是不可用的.
* iat: jwt的签发时间
* jti: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。

公共的声明：

公共的声明可以添加任何的信息，一般添加用户的相关信息或其他业务需要的必要信息.但不建议添加敏感信息，因为该部分在客户端可解密

私有的声明：

私有声明是提供者和消费者所共同定义的声明，一般不建议存放敏感信息，因为base64是对称解密的，意味着该部分信息可以归类为明文信息。

定义一个payload:

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

然后将其进行base64加密，得到Jwt的第二部分：

```
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9
```

*signature*

jwt的第三部分是一个签证信息，这个签证信息由三部分组成：

```
header (base64后的)
payload (base64后的)
secret
```

这个部分需要base64加密后的header和base64加密后的payload使用.连接组成的字符串，
然后通过header中声明的加密方式进行加盐secret组合加密，然后就构成了jwt的第三部分。

```js
// javascript
var encodedString = base64UrlEncode(header) + '.' + base64UrlEncode(payload);
var signature = HMACSHA256(encodedString, 'secret'); // TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

将这三部分用.连接成一个完整的字符串,构成了最终的jwt:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWV9.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

注意：secret是保存在服务器端的，jwt的签发生成也是在服务器端的，secret就是用来进行jwt的签发和jwt的验证，
所以，它就是你服务端的私钥，在任何场景都不应该流露出去。一旦客户端得知这个secret, 那就意味着客户端是可以自我签发jwt了。

## 如何应用

一般是在请求头里加入Authorization，并加上Bearer标注：

```js
fetch('api/user/1', {
    headers: {
        'Authorization': 'Bearer ' + token
    }
})
```

服务器负责解析这个HTTP头来做用户认证和授权处理。大致流程如下：

![](https://xnstatic-1253397658.file.myqcloud.com/jwt20.png)

## 攻击面

### 敏感信息泄露

JWT中的header 和 payload 虽然看起来不可读，但实际上都只经过简单编码，
开发者可能误将敏感信息存储在里面。使用上述工具可以方便地解码JWT中前两部分的信息。

### 指定算法为none

上面提到算法 none 是JWT规范中强制要求实现的，但有些实现JWT的库直接将使用none 算法的token视为已经过校验。
这样攻击者就可以设置alg 为none ，使signature 部分为空，然后构造包含任意payload 的JWT来欺骗服务端。

### 将签名算法从非对称类型改为对称类型

使用非对称加密算法（主要基于RSA、ECDSA，如S256）分发JWT的过程是使用私钥（private）加密生成JWT，使用公钥（public）解密验证。

使用对称加密算法（主要基于HMAC，如HS256）分发JWT的过程是使用同一个密钥（secret）生成和验证JWT。

如果服务端期待收到的算法类型为RS256，然后以RS256和public去验证JWT，而实际上收到的算法类型是HS256，
那么服务端就可能尝试把public当作secret，然后用HS256算法解密验证JWT。

由于RS256的public人人都可获得，攻击者可以预先以public为密钥，用HS256算法伪造包含任意payload 的JWT，从而成功通过服务端的验证。

### 爆破密钥

WT的安全性依赖于密钥的保密性，任何拥有密钥的人都可以构造任何内容的合法token。

当一个JSON Web Token 被分发出去，如果密钥不够强壮就存在被爆破的风险，而且整个爆破过程可以离线进行。

已经有人写了一些工具，推荐如下：

* [jwtbrute](https://github.com/jmaxxz/jwtbrute)
* [Sjord’ python script](https://github.com/Sjord/jwtcrack/blob/master/crackjwt.py)
* [John the Ripper](https://github.com/magnumripper/JohnTheRipper)

### 伪造密钥

时JWT采用header 中的kid 字段关联校验算法的密钥，这个密钥可能是对称加密的密钥，也可能是非对称加密的公钥。
如果能够猜测kid 和 密钥的关联性，攻击者就可能修改kid 来欺骗服务端，使其校验时使用攻击者可控的密钥，
于是攻击者就可以伪造任意内容的可通过校验的JWT。

## 安全建议

验证函数应忽略JWT中的algo 字段，预先就明确JWT使用的算法，如果需要使用多种算法，可以在header 中使用表示"key ID" 的kid
字段，查询每个kid 对应的算法。
JWT/JWS 标准应该移除 header 中的algo 字段。JWT的许多安全缺陷都来自于开发者依赖这一客户端可控的字段。
开发者应升级相应库到最新版本，因为旧版本可能存在致命缺陷。

1. 不应该在jwt的payload部分存放敏感信息，因为该部分是客户端可解密的部分。
1. 保护好secret私钥，该私钥非常重要。
1. 请使用https的安全通道进行传输
1. 验证函数应忽略JWT中的algo 字段，预先就明确JWT使用的算法，如果需要使用多种算法，
   可以在header中使用表示"key ID"的kid字段，查询每个kid对应的算法。
1. JWT/JWS 标准应该移除 header 中的algo 字段。JWT的许多安全缺陷都来自于开发者依赖这一客户端可控的字段。
1. 开发者应升级相应库到最新版本，因为旧版本可能存在致命缺陷。

## 参考文章

* [什么是 JWT -- JSON WEB TOKEN](https://www.jianshu.com/p/576dbf44b2ae)
* [JSON Web Token的认识与攻击](https://findneo.github.io/180503jwt/)

