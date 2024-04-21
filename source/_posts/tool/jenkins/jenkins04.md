---
title: Jenkins持续集成 - 高级特性
date: 2017-03-27 20:10:33 +0800
toc: true
categories: [ 开发工具 ]
tags: [ jenkins ]
---

这一篇记录一下Jenkins的一些有趣的东西，或者说更加接近于实战的东西，也许我写的这几篇内容只覆盖了20%左右的内容，
但是应该能解决实际工作中80%左右的问题。这就是常说的2/8准则，时间有限，我也只会去记录这些常用的东西。
<!-- more -->

## Master/Slave模式

对于大型构件项目而已，一台机器肯定是不够用的，Jenkins支持Master/Slave模式，可以添加任意多的从节点，将复杂任务分发出去。

另外对于不同的构建环境要求，比如Linux环境和Windows环境需要不同节点构建和测试，也需要设置多个从节点。

添加从节点方法：
系统管理 ->管理节点 -> 新建节点

这里我演示新建一个从节点，ip为：192.168.217.212，名称为212节点

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins32.png)

请注意有多种启动从节点的方式，这里我选择最常用的通过SSH方式启动，这里就需要配置Credentials，
通过SSH Key免密码登录从节点，用户名就是Jenkins进程启动用户tomcat（一般来讲用户名应该为jenkins）。

添加完SSH Key以后，最好在slave节点亲自测试是否能通过用户tomcat能否clone和push代码。

添加好了从节点后测试下，这里直接通过pipeline script的方式去测试，`Jenkinsfile`内容如下：

```
pipeline {
    // 默认的JENKINS_HOME只存在于master节点，/opt/tomcat/.jenkins/
    // 默认的WORKSPACE /var/jenkins/workspace/test
    agent { label 'centos7' }
    environment {
       home = "${JENKINS_HOME}"
       workspace = "${WORKSPACE}"
    }
    stages {
        stage('build') {
            steps {
                sh 'echo "test1111" > test.txt'
                sh """echo "JENKINS_HOME === ${JENKINS_HOME}" >> test.txt"""
                sh """echo "workspace === ${workspace}" >> test.txt"""
                sh 'python --version >> test.txt'
            }
        }
    }
}
```

测试截图：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins33.png)

## 主目录和工作目录

Jenkins里面有两个目录很重要，一个是主目录，环境变量访问名称为`${JENKINS_HOME}`，
默认为`/opt/tomcat/.jenkins/`，这个只在master节点上面有意义，所有jenkins配置，插件等等都放这里面。

另外一个叫工作目录，环境变量访问名称为`${WORKSPACE}`，这个在每个节点都要定义，也就是任务运行时候所处的目录，
比如源码拉下来放哪个目录。默认为`/var/jenkins/workspace`

上面我写的Jenkinsfile特地就是测试着两个变量的，可以去亲自测试一下。

