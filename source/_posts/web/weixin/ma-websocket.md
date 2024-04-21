---
title: 小程序开发 - websocket
date: 2017-12-15 12:47:12
toc: true
categories: [ web ]
tags: [ 小程序 ]
---

小程序已经添加了对WebSocket的支持，如果需要长连接和推送的场景来讲可以使用。
关于websocket的使用，[小程序WebSocket API](https://mp.weixin.qq.com/debug/wxadoc/dev/api/network-socket.html) 里边已经给了。
相比较传统的HTTP接口形式，websocket长连接可实现双向通信，但是用好它可并不简单。
<!-- more -->

刚开始的时候做这个推送，我选用了Socket.IO协议来实现，服务器端用netty-socketio实现，
而小程序用了一个第三方库[wxapp-socket-io](https://github.com/wxsocketio/wxapp-socket-io)，但是遇到各种问题，连接自动断掉，一直连不上服务器。

经常出现如下情况，后台发现连接已经断了：

![](https://xnstatic-1253397658.file.myqcloud.com/mapp01.png)

后来查询资料发现，socket.io 支持的协议版本为4，参考 [socket.io-protocol](https://github.com/socketio/socket.io-protocol)

微信小程序 websocket 协议版本为13，可以用ws包或者以他为依赖的 [中间件ws](https://github.com/websockets/ws)

后来服务器改成SpringBoot集成的WebSocket实现，实现标准协议，客户端采用小程序官方API。不过又出现了其他的问题，
踩过很多的坑，最后终于完美解决连接不稳定和断线重连的问题，本篇文章做一个总结。

注：在开发小程序长连接的过程中，在网上也google过大量的文章，但是都是一些皮毛，一个单页面演示一下连接成功例子就完了，
都是些玩具级别的例子，对于断线重连机制要么就没做，要么就做的很差。

## 小程序API介绍

具体的参数说明我就不贴出来了。

**wx.connectSocket(OBJECT)**

创建一个 WebSocket 连接。

基础库版本 1.7.0 及以后，支持存在多个 WebSokcet 连接，每次成功调用 wx.connectSocket 会返回一个新的 SocketTask。

**wx.onSocketOpen(CALLBACK)**

监听WebSocket连接打开事件。

**wx.onSocketError(CALLBACK)**

监听WebSocket错误。

**wx.sendSocketMessage(OBJECT)**

通过 WebSocket 连接发送数据，需要先 wx.connectSocket，并在 wx.onSocketOpen 回调完成之后才能发送。

**wx.onSocketMessage(CALLBACK)**

监听WebSocket接受到服务器的消息事件。

**wx.closeSocket(OBJECT)**

关闭 WebSocket 连接。

**wx.onSocketClose(CALLBACK)**

监听WebSocket关闭。

## SocketTask

这个类我要专门拿出来讲，因为小程序官方文档上面对它的介绍并不完整，导致小程序控制WebSocket的连接时遇到各种坑。

上面讲过，这个对象是通过方法`wx.connectSocket(OBJECT)`来获取的，它有一个属性值`readyState`，取下面4个状态值：

1. CONNECTING:0 连接中
1. OPEN:1 已连接
1. CLOSING:2 关闭中
1. CLOSED:3 已关闭

刚开始我们的做法是全局一个变量`socketOpen`，用来表示这个socket是否打开，没有打开就重连，否则就直接调用发送消息接口了。
但是经过测试发现网络不稳定，会出现这个变量没有得到及时更新一直是true，然后就不再去连接了，但实际上已经断开了。

所以最后把这个`socketOpen`变量去掉，直接判断SocketTask对象的属性值`readyState`，如果是1的话就表示连接可用。

## 小程序生命周期

要写好小程序应用，就必须理解小程序的生命周期，以及每个生命周期发生的时候会调用什么方法。

小程序分为应用和页面两个部分，所以小程序的生命周期就涉及到三个部分，分别是：

1. 应用的生命周期
1. 页面的生命周期
1. 应用的生命周期对页面生命周期的影响

### 应用的生命周期

App() 函数用来注册一个小程序。接受一个 object 参数，其指定小程序的生命周期函数等。

object参数说明：

 属性       | 类型        | 描述               | 触发时机                             
----------|-----------|------------------|----------------------------------
 onLaunch | Function  | 生命周期函数--监听小程序初始化 | 当小程序初始化完成时，会触发 onLaunch（全局只触发一次） 
 onShow   | Function	 | 生命周期函数--监听小程序显示  | 当小程序启动，或从后台进入前台显示，会触发 onShow     
 onHide   | Function  | 生命周期函数--监听小程序隐藏  | 当小程序从前台进入后台，会触发 onHide           

前台、后台定义：当用户点击右上角关闭，或者按了设备 Home 键离开微信，小程序并没有直接销毁，而是进入了后台；
当再次进入微信或再次打开小程序，又会从后台进入前台。

* 用户首次打开小程序，触发 onLaunch（全局只触发一次）。
* 小程序初始化完成后，触发onShow方法，监听小程序显示。
* 小程序从前台进入后台，触发 onHide方法。
* 小程序从后台进入前台显示，触发 onShow方法。
* 小程序后台运行一定时间，或系统资源占用过高，会被销毁。

![](https://xnstatic-1253397658.file.myqcloud.com/mapp02.png)

### 页面的生命周期

Page()函数用来注册一个页面。接受一个 object 参数，其指定页面的初始数据、生命周期函数、事件处理函数等。

object 参数说明：

 属性       | 类型       | 描述                 
----------|----------|--------------------
 data     | Object   | 页面的初始数据            
 onLoad   | Function | 生命周期函数--监听页面加载     
 onReady  | Function | 生命周期函数--监听页面初次渲染完成 
 onShow   | Function | 生命周期函数--监听页面显示     
 onHide   | Function | 生命周期函数--监听页面隐藏     
 onUnload | Function | 生命周期函数--监听页面卸载     

说明：

* 小程序注册完成后，加载页面，触发onLoad方法。
* 页面载入后触发onShow方法，显示页面。
* 首次显示页面，会触发onReady方法，渲染页面元素和样式，一个页面只会调用一次。
* 当小程序后台运行或跳转到其他页面时，触发onHide方法。
* 当小程序有后台进入到前台运行或重新进入页面时，触发onShow方法。
* 当使用重定向方法wx.redirectTo(OBJECT)或关闭当前页返回上一页wx.navigateBack()，触发onUnload。

![](https://xnstatic-1253397658.file.myqcloud.com/mapp03.png)

理解这些后，才能开始编写服务器和客户端代码

## 服务器端

服务器代码就不贴了，请参考我的 [github源码](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-websocket)

服务器端主要修正2个问题，一个是经常出现的Idle Timeout的异常，另外一个是Druid数据库连接池异常。

## 客户端

基本思路是：

1. 全局维护一个SocketTask对象，用来表示websocket连接，判断是否断线，作为重连的依据。
1. 同时定义一个全局callback回调函数，每个页面初始化的时候更新这个回调函数，那么在每个页面中收到返回消息就会执行当前页面逻辑。
1. 维护一个消息队列，所有消息请求会首先判断连接是否可用，如果可用直接发消息，否则将消息push到这个队列中。
1. 在`app.js`的`onShow()`函数中判断连接是否连上，如果没有连上就会触发websocket连接
1. SocketTask对象的`onOpen()`负责从消息队列中取出请求消息，并发送这个请求消息
1. SocketTask对象的`onMessage()`负责接收返回消息，并调用每个页面自己定义的回调函数
1. SocketTask对象的`onClose()`监听函数中，触发websocket连接

下面是app.js代码：

```js
let socketMsgQueue = []
let isLoading = false

App({
  globalData: {
    userInfo: null,
    localSocket: {},
    callback: function () {}
  },
  onLaunch: function (options) {
    // 展示本地存储能力
    var logs = wx.getStorageSync('logs') || []
    logs.unshift(Date.now())
    wx.setStorageSync('logs', logs)
    const updateManager = wx.getUpdateManager()
    updateManager.onUpdateReady(function () {
      updateManager.applyUpdate()
    })
    let that = this
    
    // 登录
    wx.login({
      success: res => {
        // 发送 res.code 到后台换取 openId, sessionKey, unionId
      }
    })
    // 获取用户信息
    wx.getSetting({
      success: res => {
        if (res.authSetting['scope.userInfo']) {
          // 已经授权，可以直接调用 getUserInfo 获取头像昵称，不会弹框
          wx.getUserInfo({
            success: res => {
              // 可以将 res 发送给后台解码出 unionId
              this.globalData.userInfo = res.userInfo
              // 由于 getUserInfo 是网络请求，可能会在 Page.onLoad 之后才返回
              // 所以此处加入回调以防止这种情况
              if (this.userInfoReadyCallback) {
                this.userInfoReadyCallback(res)
              }
            }
          })
        }
      }
    })
  },
  showLoad() {
    if(!isLoading) {
      wx.showLoading({
        title: '请稍后...',
      })
      isLoading = true
    }
  },
  hideLoad() {
    wx.hideLoading()
    isLoading = false
  },
  initSocket() {
    let that = this
    that.globalData.localSocket = wx.connectSocket({
      // url: 'wss://test.enzhico.net/app'
      url: 'wss://mapp.enzhico.net/app'
    })
    that.showLoad()
    that.globalData.localSocket.onOpen(function (res) {
      console.log('WebSocket连接已打开！readyState=' + that.globalData.localSocket.readyState)
      that.hideLoad()
      while (socketMsgQueue.length > 0) {
        var msg = socketMsgQueue.shift();
        that.sendSocketMessage(msg);
      }
    })
    that.globalData.localSocket.onMessage(function(res) {
      that.hideLoad()
      that.globalData.callback(res)
    })
    that.globalData.localSocket.onError(function(res) {
      console.log('readyState=' + that.globalData.localSocket.readyState)
    })
    that.globalData.localSocket.onClose(function (res) {
      console.log('WebSocket连接已关闭！readyState=' + that.globalData.localSocket.readyState)
      that.initSocket()
    })
  },
  //统一发送消息
  sendSocketMessage: function (msg) {
    if (this.globalData.localSocket.readyState === 1) {
      this.showLoad()
      this.globalData.localSocket.send({
        data: JSON.stringify(msg)
      })
    } else {
      socketMsgQueue.push(msg)
    }
  },
  onShow: function(options) {
    if (this.globalData.localSocket.readyState !== 0 && this.globalData.localSocket.readyState !== 1) {
      console.log('开始尝试连接WebSocket！readyState=' + this.globalData.localSocket.readyState)
      this.initSocket()
    }
  }
})
```

下面是某个页面的onshow()函数：

```js
/**
* 生命周期函数--监听页面显示
*/
onShow: function () {
  var that = this
  app.globalData.callback = function (res) {
    let resData = JSON.parse(res.data)
    let data = resData.result
    if (resData.method == 'list') {
      that.setData({
        total: parseInt(data.allAmount),
        amount: data.orderNum,
        jdcAmount: parseInt(data.jdcAmount),
        jszAmount: parseInt(data.jszAmount),
        list: data.list || []
      })
      if(data.list == 0) {
        that.setData({
          isEmpty: true
        })
      }
      that.getNumber(resData.result.allAmount)
      that.setAmount()
    } else if (resData.method == 'notify') {
      // 服务器推送消息，省略具体逻辑
    }
  }
  setTimeout(function () {
    app.sendSocketMessage({
      method: 'list'
    })
  }, 300)
},
```

