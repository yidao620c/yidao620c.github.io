---
title: 【安全贴士】HTTPS协议详解
date: 2017-11-16 18:22:19 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 数字证书
  - 安全贴士
---

HTTPS（全称：`Hypertext Transfer Protocol over Secure Socket Layer`），是以安全为目标的HTTP通道，简单讲是HTTP的安全版。
即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。

HTTPS的优点：

1. 内容加密建立一个信息安全通道，来保证数据传输的安全；
1. 身份认证确认网站的真实性
1. 数据完整性防止内容被第三方冒充或者篡改

当然凡事都有两面性，HTTPS的缺点也很明显，就是处理速度会比HTTP慢，另外网站的CA证书申请一般都需要一笔费用。
<!-- more -->

## 加解密相关知识

### 对称加密

对称加密指加密和解密使用相同密钥的加密算法，有时又叫传统密码算法，就是加密密钥能够从解密密钥中推算出来，
同时解密密钥也可以从加密密钥中推算出来。而在大多数的对称算法中，加密密钥和解密密钥是相同的，
所以也称这种加密算法为秘密密钥算法或单密钥算法。

常见的对称加密有：DES（Data Encryption Standard）、AES（Advanced Encryption Standard）、RC4、IDEA

### 非对称加密

与对称加密算法不同，非对称加密算法需要两个密钥：公开密钥（publickey）和私有密钥（privatekey）；
并且加密密钥和解密密钥是成对出现的。非对称加密算法在加密和解密过程使用了不同的密钥，非对称加密也称为公钥加密，
在密钥对中，其中一个密钥是对外公开的，所有人都可以获取到，称为公钥，其中一个密钥是不公开的称为私钥。

非对称加密算法对加密内容的长度有限制，不能超过公钥长度。比如现在常用的公钥长度是 2048 位，意味着待加密内容不能超过 256 个字节。

### 摘要算法

数字摘要是采用单项Hash函数将需要加密的明文"摘要"成一串固定长度（128位）的密文，这一串密文又称为数字指纹，
它有固定的长度，而且不同的明文摘要成密文，其结果总是不同的，而同样的明文其摘要必定一致。
数字摘要是https能确保数据完整性和防篡改的根本原因。

### 数字签名

数字签名技术就是对"非对称密钥加解密"和"数字摘要"两项技术的应用，它将摘要信息用发送者的私钥加密，与原文一起传送给接收者。
接收者只有用发送者的公钥才能解密被加密的摘要信息，然后用HASH函数对收到的原文产生一个摘要信息，与解密的摘要信息对比。
如果相同，则说明收到的信息是完整的，在传输过程中没有被修改，否则说明信息被修改过，因此数字签名能够验证信息的完整性。

数字签名的过程如下：

```
明文 --> hash运算 --> 摘要 --> 私钥加密 --> 数字签名
```

数字签名有两种功效：

1. 能确定消息确实是由发送方签名并发出来的，因为别人假冒不了发送方的签名。
2. 数字签名能确定消息的完整性。

注意：数字签名只能验证数据的完整性，数据本身是否加密不属于数字签名的控制范围

### 数字证书

**为什么要有数字证书？**

对于请求方来说，它怎么能确定它所得到的公钥一定是从目标主机那里发布的，而且没有被篡改过呢？
亦或者请求的目标主机本本身就从事窃取用户信息的不正当行为呢？
这时候，我们需要有一个权威的值得信赖的第三方机构(一般是由政府审核并授权的机构)来统一对外发放主机机构的公钥，
只要请求方这种机构获取公钥，就避免了上述问题的发生。

**数字证书的颁发过程**

用户首先产生自己的密钥对，并将公共密钥及部分个人身份信息传送给认证中心。认证中心在核实身份后，将执行一些必要的步骤，
以确信请求确实由用户发送而来，然后，认证中心将发给用户一个数字证书，该证书内包含用户的个人信息和他的公钥信息，
同时还附有认证中心的签名信息(根证书私钥签名)。用户就可以使用自己的数字证书进行相关的各种活动。
数字证书由独立的证书发行机构发布，数字证书各不相同，每种证书可提供不同级别的可信度。

**证书包含哪些内容**

* 证书颁发机构的名称
* 证书本身的数字签名
* 证书持有者公钥
* 证书签名用到的Hash算法

**验证证书的有效性**

浏览器默认都会内置CA根证书，其中根证书包含了CA的公钥

1、证书颁发的机构是伪造的：浏览器不认识，直接认为是危险证书

2、证书颁发的机构是确实存在的，于是根据CA名，找到对应内置的CA根证书、CA的公钥。用CA的公钥，
对伪造的证书的摘要进行解密，发现解不了，认为是危险证书。

3、对于篡改的证书，使用CA的公钥对数字签名进行解密得到摘要A，然后再根据签名的Hash算法计算出证书的摘要B，
对比A与B，若相等则正常，若不相等则是被篡改过的。

4、证书可在其过期前被吊销，通常情况是该证书的私钥已经失密。较新的浏览器如Chrome、Firefox、Opera和Internet Explorer
都实现了在线证书状态协议（OCSP）以排除这种情形：浏览器将网站提供的证书的序列号通过OCSP发送给证书颁发机构，
后者会告诉浏览器证书是否还是有效的。

1、2点是对伪造证书进行的，3是对于篡改后的证书验证，4是对于过期失效的验证。

## SSL 与 TLS

SSL (Secure Socket Layer，安全套接字层)

SSL为Netscape所研发，用以保障在Internet上数据传输之安全，利用数据加密(Encryption)技术，
可确保数据在网络上之传输过程中不会被截取，当前为3.0版本。

TLS (Transport Layer Security，传输层安全协议)

TLS 1.0是IETF（Internet Engineering Task Force，Internet工程任务组）制定的一种新的协议，
它建立在SSL 3.0协议规范之上，是SSL 3.0的后续版本，可以理解为SSL 3.1

简单来讲，HTTPS 是身披 SSL/TLS 外壳的 HTTP，或者说：HTTP + 加密 + 认证 + 完整性保护 = HTTPS。

请注意，SSL/TSL跟HTTP没有半毛钱的关系，它就是一件安全的外套而已，
被HTTP披上之后变成HTTPS，被FTP披上之后就变成FTPS了。所以很多应用层协议披上这层外套就会有了一个安全的保障。

## SSL/TLS的握手过程

SSL与TLS握手整个过程如下图所示，下面会详细介绍每一步的具体内容：

![](https://xnstatic-1253397658.file.myqcloud.com/https01.png)

详细步骤讲解

### 客户端首次发出请求

客户端需要提供如下信息：

* 支持的协议版本，比如TLS 1.0版
* 一个客户端生成的随机数，稍后用于生成"对话密钥"
* 支持的加密算法，比如RSA公钥加密
* 支持的压缩方法

### 服务端首次回应

服务端在接收到客户端的Client Hello之后，服务端需要确定加密协议的版本，以及加密的算法，然后也生成一个随机数，
以及将自己的证书发送给客户端一并发送给客户端，这里的随机数是整个过程的第二个随机数。

服务端需要提供的信息：

* 协议的版本
* 加密算法
* 随机数
* 服务器证书

### 客户端再次回应

客户端首先会对服务器下发的证书进行验证，验证通过之后，则会继续下面的操作，客户端再次产生一个随机数（第三个随机数），
然后使用服务器证书中的公钥进行加密，以及放一个ChangeCipherSpec消息，还有整个前面所有消息的hash值，
然后用新秘钥加密一段数据一并发送到服务器，确保正式通信前无误。

客户端使用前面的两个随机数以及刚刚新生成的新随机数，使用与服务器确定的加密算法，生成一个`Session Secret`。

### 服务器再次响应

服务端在接收到客户端传过来的第三个随机数的 加密数据之后，使用私钥对这段加密数据进行解密，并对数据进行验证，
也会使用跟客户端同样的方式生成秘钥，一切准备好之后，也会给客户端发送一个 ChangeCipherSpec，
告知客户端已经切换到协商过的加密套件状态，准备使用加密套件和 `Session Secret` 加密数据了。
之后，服务端也会使用 `Session Secret` 加密一段 Finish 消息发送给客户端，以验证之前通过握手建立起来的加解密通道是否成功。

### 后续客户端与服务器间通信

确定秘钥之后，服务器与客户端之间就会通过商定的秘钥 `Session Secret` 加密消息进行通讯了，整个握手过程也就基本完成了。

另外还有一个更简洁直观的握手图：

![](https://xnstatic-1253397658.file.myqcloud.com/https02.png)

## 总结

相比 HTTP 协议，HTTPS 协议增加了很多握手、加密解密等流程，虽然过程很复杂，但其可以保证数据传输的安全。
所以在这个互联网膨胀的时代，其中隐藏着各种看不见的危机，为了保证数据的安全，维护网络稳定，建议大家多多推广HTTPS。

*参考文章*

* [Https协议详解](https://www.jianshu.com/p/30b8b40a671c)
* [HTTPS系列干货（一）：HTTPS 原理详解](https://zhuanlan.zhihu.com/p/27395037)

