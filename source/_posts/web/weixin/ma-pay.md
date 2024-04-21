---
title: 小程序开发 - 跳转支付
date: 2017-12-13 09:30:22
toc: true
categories: [ web ]
tags: [ 小程序 ]
---

根据微信支付的官方文档，小程序支付需要绑定商户号，并且小程序和商户号的认证主体必须一致。
目前我们的商业逻辑是小程序平台主体和支付的商户主体不一致，
那么就需要从我们的小程序跳转到支付主体小程序完成支付后，再返回我们的小程序。

曾经尝试过在小程序中通过webview的方式嵌套H5网页，使用公众号支付方式，后来发现小程序并未开放这个JSAPI。

接下来详细介绍一下如何实现小程序之间的跳转支付。

首先我已经有了2个小程序：

1. 平台小程序A，认证主体是A公司
2. 支付小程序B，认证主体是B公司

<!-- more -->

## 接入微信支付

支付小程序B要接入微信支付，必须要先注册并认证微信公众号，然后申请服务商商户，
具体怎么申请参考[微信支付服务商平台](https://pay.weixin.qq.com/partner/public/home)

然后登录[小程序后台](mp.weixin.qq.com)，点击左侧导航栏的微信支付，在页面中进行开通。

![](https://xnstatic-1253397658.file.myqcloud.com/ma001.png)

点击开通按钮后，有2种方式可以获取微信支付能力，新申请微信支付商户号或绑定一个已有的微信支付商户号，
请根据你的业务需要和具体情况选择，只能二选一。

开通指引：<http://kf.qq.com/faq/140225MveaUz161230yqiIby.html>

小程序支付开发注意点：

* appid 必须为最后拉起收银台的小程序appid；
* mch_id 为和appid成对绑定的支付商户号，收款资金会进入该商户号；
* trade_type 请填写JSAPI；
* openid 为appid对应的用户标识，即使用wx.login接口获得的openid

## 小程序跳转

参考官方文档[打开小程序](https://mp.weixin.qq.com/debug/wxadoc/dev/api/navigateToMiniProgram.html)

`wx.navigateToMiniProgram(OBJECT)`，打开同一公众号下关联的另一个小程序。（注：必须是同一公众号下，而非同个open账号下）

所以第一步要将两个小程序关联到同一个公众号上面去，这里一般都会关联到平台的公众号上去。
关联操作需要从公众号发起，请登录公众平台上公众号的管理后台，在"设置-公众号设置-相关小程序"中进行关联。

然后小程序的管理员会受到一条微信消息，点击进去同意即可。之后再进入小程序后台，就能看到小程序所关联的公众号了：

![](https://xnstatic-1253397658.file.myqcloud.com/ma003.png)

小程序A跳转到B的示例代码：

```js
wx.navigateToMiniProgram({
    appId: 'APPID',
    path: 'pages/index/index?id=123',
    extraData: {
        foo: 'bar'
    },
    envVersion: 'develop',
    success(res) {
        // 打开成功
    }
})
```

同时还可以给小程序B传递参数，通过`extraData`这个参数，小程序B接受参数的方法：

```js
App({
  onLaunch: function(options) {
    wx.setStorageSync('fromAppId', options.referrerInfo.appId)
    wx.setStorageSync('foo', options.referrerInfo.extraData.foo)
  },
  onShow: function(options) {
      // Do something when show.
  },
  onHide: function() {
      // Do something when hide.
  },
  onError: function(msg) {
    console.log(msg)
  },
  globalData: 'I am global data'
})
```

小程序B处理完成后返回小程序A的方式：

```js
console.log('支付成功啦啦啦啦。。。。。')
wx.navigateBackMiniProgram({
    extraData: {
        payResult: 'good'
    },
    success(res) {
        // 打开成功
    }
})
```

然后小程序A在`App.js`中接受到返回值：

```js
onShow: function (options) {
    wx.setStorageSync('payResult', options.referrerInfo.extraData.payResult)
}
,
```

然后在相应的页面就能处理结果值了，这里还是用onShow函数：

```js
onShow: function (options) {
    this.setData({
        'lalala': wx.getStorageSync('payResult')
    })
}
,
```

## 小程序支付

小程序B需要实现微信支付功能，还需要有一个后端系统，我这里通过一个SpringBoot工程搭建后端系统，
另外引入一个微信支付的依赖：

```xml
<dependency>
    <groupId>com.github.binarywang</groupId>
    <artifactId>weixin-java-pay</artifactId>
    <version>${weixin-java-pay.version}</version>
</dependency>
```

主要的逻辑是这样：

1. 小程序向服务端发送商品详情、金额、openid
1. 服务端向微信统一下单
1. 服务器收到返回信息二次签名发回给小程序
1. 小程序发起支付
1. 服务端收到回调

这里需要注意的是，小程序调用后台接口的地址需要在小程序后台配置好服务器域名：

![](https://xnstatic-1253397658.file.myqcloud.com/ma002.png)

服务器域名只能是https开头，所以需要先去给你的网站申请一个SSL证书，配置一下nginx来代理转发即可，
比如我的域名为`https://aggrepay.enzhico.cn/`，这里我申请的是lets encrypted证书：

```nginx
server {
    listen 443 ssl;
    server_name aggrepay.enzhico.cn;

    location / {
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header 'Access-Control-Allow-Origin' '*';
        proxy_set_header 'Access-Control-Allow-Credentials' 'true';
        proxy_set_header 'Access-Control-Allow-Headers' 'Authorization,Content-Type,Accept,Origin,User-Agent,DNT,Cache-Control,X-Mx-ReqToken,X-Requested-With';
        proxy_set_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS';
        proxy_pass http://119.29.12.177:8086/;
    }

    # letsencrypt生成的文件
    ssl_certificate /etc/letsencrypt/live/aggrepay.enzhico.cn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/aggrepay.enzhico.cn/privkey.pem;

    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets on;

    ssl_dhparam /etc/ssl/private/dhparam.pem;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    # 一般推荐使用的ssl_ciphers值: https://wiki.mozilla.org/Security/Server_Side_TLS
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128:AES256:AES:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK';
    ssl_prefer_server_ciphers on;
}
```

设置完服务器后，就将后台应用部署上去。

回到小程序B中，先要获取到用户openid，小程序是通过登录操作获取到用户的openid的。

参考官方的[小程序登录API](https://mp.weixin.qq.com/debug/wxadoc/dev/api/api-login.html)

很重要的一张登录时序图：

![](https://xnstatic-1253397658.file.myqcloud.com/ma004.png)

```js
  onLoad: function () {
    var that = this
    //登陆获取code
    wx.login({
        success: function (res) {
            console.log("code = " + res.code)
            //获取openid
            that.getOpenId(res.code)
        }
    });
}
,
getOpenId: function (code) {
    var that = this;
    wx.request({
        url: "https://aggrepay.enzhico.cn/pay/openid?code=" + code,
        data: {},
        method: 'GET',
        success: function (res) {
            console.log("res.openid = " + res.data.openid)
            that.generateOrder(res.data.openid)
        },
        fail: function () {
            // fail
        },
        complete: function () {
            // complete
        }
    })
}
,
```

对应后台代码：

```java
/**
 * get openid
 *
 * @return 成功标准
 */
@RequestMapping(value = "/openid", method = RequestMethod.GET)
@ResponseBody
public String openid(@RequestParam(value = "code") String code) throws Exception {
    logger.info("/openid start....");
    CloseableHttpClient httpclient = HttpClients.createDefault();
    HttpGet httpGet = new HttpGet("https://api.weixin.qq.com/sns/jscode2session?" +
            "appid=APPID&secret=SECRET&js_code=" + code + "&grant_type=authorization_code");
    try (CloseableHttpResponse response1 = httpclient.execute(httpGet)) {
        logger.info(response1.getStatusLine().toString());
        String result = EntityUtils.toString(response1.getEntity(), StandardCharsets.UTF_8);
        logger.info("result=" + result);
        return result;
    }
}
```

获取到openid后就可以调用统一下单接口了：

```js
  /**生成商户订单 */
generateOrder: function (openid) {
    var that = this
    //统一支付
    wx.request({
        url: 'https://aggrepay.enzhico.cn/pay/unifiedOrder',
        method: 'POST',
        data: {
            outTradeNo: Math.random().toString(36).substring(7),
            totalFee: '7',
            body: '支付测试',
            attach: '云塔山香烟',
            spbillCreateIp: '123.267.12.2',
            notifyURL: 'https://aggrepay.enzhico.cn/pay/parseOrderNotifyResult',
            tradeType: 'JSAPI',
            openid: openid
        },
        success: function (res) {
            //发起支付
            var timeStamp = res.data.timeStamp;
            var packages = 'prepay_id=' + res.data.prepayId;
            var paySign = res.data.sign;
            var nonceStr = res.data.nonceStr;
            var param = {
                "timeStamp": timeStamp,
                "package": packages,
                "paySign": paySign,
                "signType": "MD5",
                "nonceStr": nonceStr
            };
            that.pay(param)
        },
    })
}
,
```

对应后台代码，注意二次签名：

```java
/**
 * 统一下单(详见https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=9_1)
 * 在发起微信支付前，需要调用统一下单接口，获取"预支付交易会话标识"
 * 接口地址：https://api.mch.weixin.qq.com/pay/unifiedorder
 *
 * @param request 请求对象，注意一些参数如appid、mchid等不用设置，方法内会自动从配置对象中获取到（前提是对应配置中已经设置）
 */
@PostMapping("/unifiedOrder")
public WxPayMpOrderResultVO unifiedOrder2(@RequestBody WxPayUnifiedOrderRequest request) throws WxPayException {
    WxPayUnifiedOrderResult result =  this.wxService.unifiedOrder(request);
    logger.info(result.getReturnMsg());
    String signType = WxPayConstants.SignType.MD5;
    WxPayMpOrderResultVO payResult = new WxPayMpOrderResultVO();
    payResult.setAppId(result.getAppid());
    payResult.setTimeStamp(String.valueOf(System.currentTimeMillis() / 1000));
    payResult.setNonceStr(result.getNonceStr());
    payResult.setPackageValue("prepay_id=" + result.getPrepayId());
    payResult.setSignType(signType);
    payResult.setPrepayId(result.getPrepayId());
    // 二次签名
    payResult.setSign(createSign(payResult, this.wxService.getConfig().getMchKey()));
    return payResult;
}

/**
 * 下面是我自己写的签名，最好调用框架的签名方法
 */
private String createSign(WxPayMpOrderResultVO p, String mchKey) {
    String tosstr = "appId="+p.getAppId()+"&nonceStr="+p.getNonceStr()+"&package="+p.getPackageValue()
            +"&signType=MD5&timeStamp="+p.getTimeStamp()+"&key=" + mchKey;
    logger.info(tosstr);
    return DigestUtils.md5Hex(tosstr).toUpperCase();
}
```

拿到预支付订单后，小程序B就能调用支付了，支付成功后返回到小程序A中，并将支付结果通过`extraData`参数回传过去：

```js
/* 支付   */
pay: function (param) {
    console.log("支付")
    console.log(param)
    wx.requestPayment({
        'timeStamp': param.timeStamp,
        'nonceStr': param.nonceStr,
        'package': param.package,
        'signType': param.signType,
        'paySign': param.paySign,
        success: function (res) {
            console.log('支付成功啦啦啦啦。。。。。')
            wx.navigateBackMiniProgram({
                extraData: {
                    payResult: 'good'
                },
                success(res) {
                    // 打开成功
                }
            })
        },
        fail: function (res) {
            // fail
        },
        complete: function () {
            // complete
        }
    })
}
```

小程序的跳转需要上线版本才能看到效果，所以需要把代码上传后，提交上线审核。
登录小程序后台，点击"开发管理"，在下面的开发版本中点击提交审核，一般半天左右审核完成。

![](https://xnstatic-1253397658.file.myqcloud.com/ma005.png)

