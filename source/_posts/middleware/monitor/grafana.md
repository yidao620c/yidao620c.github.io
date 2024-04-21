---
title: Grafana快速入门
date: 2021-02-05 09:12:33 +0800
toc: true
categories: [ 中间件 ]
tags: [ grafana ]
---

Grafana 是一个监控仪表系统，它是由 Grafana Labs 公司开源的的一个系统监测 (System Monitoring) 工具。
它可以大大帮助你简化监控的复杂度，你只需要提供你需要监控的数据，它就可以帮你生成各种可视化仪表。
同时它还有报警功能，可以在系统出现问题时通知你。

这个是它的主界面：
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-01.png)

Grafana 不对数据源作假设，它支持以下各种数据，也就是说如果你的数据源是以下任意一种，它都可以帮助生成仪表。
同时在市面上，如果 Grafana 称第二，那么应该没有敢称第一的仪表可视化工具了。因此，如果你搞定了 Grafana，
它几乎是一个会陪伴你到各个公司的一件称心应手的兵器。

Grafana 支持的数据源：Prometheus、Graphite、OpenTSDB、InfluxDB、MySQL/PostgreSQL、
Microsoft SQL Server、等等。

<!-- more -->

**什么情况下会用到 Grafana 或者监控仪表盘**

通常来说，对于一个运行时的复杂系统，你是不太可能在运行时一边检查代码一边调试的。因此，你需要在各种关键点加上监控。

用开车作为例子：车子本身是一个极其复杂的系统，而当你的车在高速上以 120 公里的速度狂奔时出现了噪音，
你是不可能这时候边开车边打开发动机盖子来查原因的。通常来说，好一点的车会有内置电脑，在车子出问题时，
告诉你左边轮胎胎压有问题，或是发动机缺水了之类。而这些检测，就是系统监控的一个例子。

仪表盘应用极广，我能想到的一些例子：

* 阿里在双十一控制室用了监控仪表盘，因此所有双十一的新闻基本上都可以看到这个仪表盘
* 各酷炫公司大厅里常常放一个仪表盘来展示实力（用户数啦、营收啦之类）
* 你的 PC 上的资源管理器、Mac 上的 Activity Monitor 都是某种意义上的仪表盘

## 安装 Grafana
为了简化各种系统不一致的乱七八糟问题，我们用 docker-compose 来安装 Grafana。
最新配置归档地址： <https://github.com/Kalasearch/grafana-tutorial>

docker-compose.yml配置如下：
```yaml
version: '3.4'
services:
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    hostname: prometheus
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
  prometheus-exporter:
    image: prom/node-exporter
    container_name: service
    hostname: service
    ports:
      - 9100:9100
  grafana:
    image: grafana/grafana
    container_name: grafana
    hostname: grafana
    ports:
      - 3000:3000
    volumes:  
      - ./grafana.ini:/etc/grafana/grafana.ini
```

在这里我们启动了三个服务

* Prometheus 普罗米修斯时序数据库，用来存储和查询你的监控数据
* Promethues-exporter 一个模拟数据源，用来监控你本机的状态，比如有几个 CPU，CPU 的负载之类
* Grafana 本尊

在 clone 了代码之后，在你的本地运行 `docker-compose up`启动即可。

跑起来服务之后，到你的浏览器中，复制 `http://IP:3000` 应该就可以看到 Grafana 跑起来的初始登录界面。
初始的用户名是 admin，密码也是 admin。输入之后，会要求你改密码。

到这里，你的 Grafana 就已经搭起来了。注意到 Docker 的配置文件中我们创建了三个服务，这三个服务之间分别有什么关系呢？
或者说，Grafana 和时序数据库，数据源之间有什么关系呢？请看下文 Grafana 工作原理

## Grfana 工作原理

上面说到，Grafana 是一个仪表盘，而仪表盘必然是用来显示数据的。

Grafana 本身并不负责数据层，它只提供了通用的接口，让底层的数据库可以把数据给它。而我们起的另一个服务，
叫 Prometheus （中文名普罗米修斯数据库）则是负责存储和查询数据的。

也就是说，Grafana 每次要展现一个仪表盘的时候，会向 Prometheus 发送一个查询请求。

那么配置里的另一个服务 Prometheus-exporter 又是什么呢？

这个就是你真正监测的数据来源了，Prometheus-exporter 这个服务，会查询你的本地电脑的信息，
比如内存还有多少、CPU 负载之类，然后将数据导出至普罗米修斯数据库。

在真实世界中，你的目的是监控你自己的服务，比如你的 Web 服务器，你的数据库之类。

那么你就需要在你自己的服务器中把数据发送给普罗米修斯数据库。当然，你完全可以把数据发送给 MySQL (Grafana 也支持)，
但Prometheus几乎是标配的时序数据库，强烈建议你用。

## 搭建第一个仪表盘
**第 1 步 - 设置数据源**

进入 Grafana 后，在左侧你会发现有一个 Data Source 即数据源选项。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-02.png)

点击后进入，点 Add Data Source 即添加数据源，选择 Prometheus
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-03.png)

之后设置数据源 URL。请注意，Promethues 的工作原理（下一个教程中会讲）是通过轮询一个 HTTP 请求来获取数据的，
而 Grafana 在获取数据源的时候也是通过一个 HTTP 请求，因此这个地方你需要告诉 Grafana 你的 Prometheus 的数据端点是什么。

这里我们填入 `http://prometheus:9090` 就可以了。

> [!NOTE]
> 你可能会问，为什么不是 localhost:9090 呢？
> 
> 原因是我们用了 docker-compose 起的三个服务，可以把它们想象成三台独立的服务器，因此需要用一个域名来互相通信。
> 我们在 docker-compose.yml 中设置的普罗米修斯服务器的名字就叫 prometheus，因此这里需要用前者。

设置好之后，点击 `Save & Test` 保存并验证。一定要确认出现 `Data source is working` 这个检测，这时表明连接成功。

**第 2 步 - 导入 Dashboard**

在 Grafana 里，仪表盘的配置可以通过图形化界面进行，但配置好的仪表盘是以 JSON 存储的。这也就是说，
如果你把你的 JSON 数据分享出去，别人导入就可以直接导入同样的仪表盘（前提是你们的监测数据一样）。

对于我们的例子来说，回忆一下，因为我们用了 prometheus-exporter 也就是本机的系统信息监控，
那么我们可以先找一个同样用了这个数据源的仪表盘。在 Grafana 网站上，你其实可以找到很多别人已经做好的仪表，
可以用来监测非常多标准化的服务。

Grafana 的仪表盘市场：<https://grafana.com/grafana/dashboards>

针对一些服务的标准仪表盘就可以在这里找到，比如JVM、Spring Boot、MySQL监控、Docker监控等等。

那么，这里我们就用一个标准的仪表盘：https://grafana.com/grafana/dashboards/1860

在左侧的加号里，点 Import 即导入，在出现的界面中填入 1860 即我们要导入的仪表盘编号即可。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-04.png)

然后填入你需要的信息，比如仪表盘名字、数据源信息等，这里选择我们刚刚配置好的Prometheus数据源即可。
确认之后 Grafana 就会根据你的本机信息，生成类似 CPU 负载，内存和 I/O 之类的信息。

要注意的是，这里的信息真正监控的是你的 Docker 中的系统信息。如果你只给你的 Docker 分配 1 个核和 2G 内存，
那么这里应该看到的就是 1 个核和 2G 内存。

**第 3 步 - 生成和创建新的仪表盘**

在上面导入信息的基础上，你就可以开始创建和你的服务、业务相关的仪表盘了。

但在这步之前，你需要先在你的服务中开始记录一些数据。

假设你已经按上面的步骤生成了一个基本的仪表盘，那么现在可以开始手动添加仪表盘了。同样是点左侧的加号，
点 Dashboard 就可以进入添加仪表盘的界面。选择一个查询叫 scrape_duration_seconds。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-05.png)

## 关于自动刷新
刚开始我创建仪表盘的时候，无法自动刷新。而且还出现不了时间选择器右边的那个自动刷新的下拉框。
原因是面板Panel是没有这个设置的，在面板中只能手动刷新。但是仪表盘Dashboard是有这个设置的。
在仪表盘的设置中，请确保Hide time picker不要选中。
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230715-06.png)

