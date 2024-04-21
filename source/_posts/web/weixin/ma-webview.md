---
title: 小程序开发 - webview
date: 2017-12-12 10:16:53
toc: true
categories: [ web ]
tags: [ 小程序 ]
---

最近要做一个项目需要在小程序中打开外链，小程序最近开放了web-view组件，
可在里面内嵌自己写的H5页面，也就实现了打开外链的功能，但是有几个注意点。
这里记录一下，希望将来小程序能放开更多限制。
<!-- more -->

## 申请业务域名

首先必须在小程序后台配置业务域名，并且是已经备案过的。

## 微信授权登录

当需要微信授权登录的H5页面直接通过小程序webview访问时，会报错。

解决方案：

对浏览器进行判断，如果是小程序webview（官方判断条件：`window.__wxjs_environment === 'miniprogram'`）就跳过授权登录。
这样就规避了访问非授权业务域名问题。

## 打开网页条件

1. 小程序基础库版本要大于 1.6.4，低版本的小程序需要做兼容处理
2. 网页内容只能在<web-view/>组件中显示，它会自动铺满整个小程序页面
3. 个人类型与海外类型的小程序暂不支持使用web-view组件打开网页

## 示例代码

```html
<!– wxml –>
<!– 指向微信公众平台首页的web-view –>
<web-view src="https://mp.weixin.qq.com/"></web-view>
```

## web-view组件接口

<web-view/>网页中可使用JSSDK 1.3.0提供的接口返回小程序页面，支持的接口有：

 接口名	                        |            | 说明最低版本 
-----------------------------|------------|--------
 wx.miniProgram.navigateTo   | 参数与小程序接口一致 | 1.6.4  
 wx.miniProgram.navigateBack | 参数与小程序接口一致 | 1.6.4  
 wx.miniProgram.switchTab    | 参数与小程序接口一致 | 即将开放   
 wx.miniProgram.reLaunch     | 参数与小程序接口一致 | 即将开放   
 wx.miniProgram.redirectTo   | 参数与小程序接口一致 | 即将开放   

示例代码：

```html
<!-- html -->
<script type="text/javascript" src="https://res.wx.qq.com/open/js/jweixin-1.3.0.js"></script>

// javascript
wx.miniProgram.navigateTo({url: '/path/to/page'})
```

<web-view/>
网页中仅支持以下JSSDK接口有限，详细请参考[官网文档说明](https://mp.weixin.qq.com/debug/wxadoc/dev/component/web-view.html)

## 其他

1. 网页内iframe的域名也需要配置到域名白名单。
1. 开发者工具上，可以在 <web-view/> 组件上通过右键 - 调试，打开 <web-view/> 组件的调试。
1. 每个页面只能有一个<web-view/>，<web-view/>会自动铺满整个页面，并覆盖其他组件。
1. <web-view/>网页与小程序之间不支持除JSSDK提供的接口之外的通信。
1. 在iOS中，若存在JSSDK接口调用无响应的情况，可在<web-view/>的src后面加个`#wechat_redirect`解决。
