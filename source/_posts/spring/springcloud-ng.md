---
title: 下一代SpringCloud技术栈
date: 2020-02-10 10:15:22 +0800
comments: true
toc: true
categories: spring
tags:
  - springcloud
---

目前市场上主流的 第一套微服务架构解决方案：Spring Boot + Spring Cloud Netflix。
最近由于Netflix公司宣布Spring Cloud Netflix 系列技术栈进入维护模式，
于是采用 Spring Cloud Alibaba 方案来替代。

我们建议对这些模块提供的功能进行以下替换

CURRENT	                  | REPLACEMENT
------------------------- | --------------------------------------
Hystrix	                  | rResilience4j/Sentinel
Hystrix Dashboard/Turbine | Micrometer + Monitoring System
Ribbon                    | Spring Cloud Loadbalancer
Zuul 1                    | Spring Cloud Gateway
Eureka                    | Nacos

出自[【官方新闻】Spring Cloud Greenwich.RC1 available now](https://spring.io/blog/2018/12/12/spring-cloud-greenwich-rc1-available-now)
<!-- more -->



