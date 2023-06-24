---
title: JVM应用度量框架Micrometer
date: 2020-04-12 23:06:23 +0800
comments: true
toc: true
categories: [开发工具]
tags:
  - micrometer
---

通常我们需要在生产环境中监控机器的性能指标和业务数据指标，一般我们叫这样的操作为"埋点"。
spring-actuator是用来做度量统计收集的，底层使用Prometheus进行数据收集，Grafana进行数据展示。
它所集成的度量统计API使用的框架叫Micrometer，官网地址为<https://micrometer.io/>。
本篇将介绍一下这个框架的基本使用方法。

<!-- more -->


