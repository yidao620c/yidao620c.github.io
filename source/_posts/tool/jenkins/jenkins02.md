---
title: Jenkins持续集成 - 管道详解
date: 2017-03-22 09:32:16 +0800
toc: true
categories: [ 开发工具 ]
tags: [ jenkins ]
---

前面一篇介绍了Jenkins的入门安装和简单演示，这篇讲解最核心的Pipeline部分。

Jenkins Pipeline 就是一系列的插件集合，可通过组合它们来实现持续集成和交付的功能。
通过`Pipeline DSL`为我们提供了一个可扩展的工具集，将简单到复杂的逻辑通过代码实现。

通常，我们可以通过编写`Jenkinsfile`将管道代码化，并且纳入到版本管理系统中。
<!-- more -->

```
// Declarative //
pipeline {
    agent any ①

    stages {
        stage('Build') { ②
            steps { ③
                sh 'make' ④
            }
        }
        stage('Test'){
            steps {
                sh 'make check'
                junit 'reports/**/*.xml' ⑤
            }
        }
        stage('Deploy') {
            steps {
                sh 'make publish'
            }
        }
    }
}

// Script //
node {
    stage('Build') {
        sh 'make'
    }
    stage('Test') {
        sh 'make check'
        junit 'reports/**/*.xml'
    }
    stage('Deploy') {
        sh 'make publish'
    }
}
```

* ① agent 指示Jenkins分配一个执行器和工作空间来执行下面的Pipeline
* ② stage 表示这个Pipeline的一个执行阶段
* ③ steps 表示在这个stage中每一个步骤
* ④ sh 执行指定的命令
* ⑤ junit 是插件`junit[JUnit plugin]`提供的一个管道步骤，用来收集测试报告

## 管道名词

几个重要的名词，讲一下它们是什么意思：

* Step 一个简单的执行步骤，比如执行一个sh脚本
* Stage 将你的命令组织成一个更高一层的逻辑单元
* Node 指定这些任务在哪执行

Stage和Step可以放到一个Node下面执行，不指定就默认在master节点上面执行。
另外Node和Step也能组合成一个Stage。

## 定义管道

有两种定义管道的方式，一种是通过Web UI来定义，一种是直接写`Jenkinsfile`。推荐后面一种，因为可以纳入版本管理系统。

### Web UI方式

这里先介绍第一种方式，通过Web UI，首先点击"新建"：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins07.png)

填写一个名字，然后选择`Pipeline`

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins08.png)

点击"OK"后，在`Script`中写一个简单的命令：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins09.png)

保存后，点击左侧的"立即构建"：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins10.png)

然后在"Build History"下面点击"#1"进入此次构建详情，再点击左侧的"Console Output"查看输出：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins11.png)

我们看到了打印出来的"hello world"说明成功运行。

上面的例子演示了通过Web UI创建的一个最基本的管道执行成功的案例。使用2个步骤：

```
// Script //
node { ①
    echo 'Hello World' ②
}
// Declarative not yet implemented //
```

* ① node 在Jenkins环境中分配一个执行器和工作空间
* ② echo 在控制台输出一个简单的字符串

### Jenkinsfile方式

上面通过Web UI方式只适用于非常简单的任务，而大型复杂的任务最好采用`Jenkinsfile`方式并纳入SCM管理。
这次我选择从SCM中的`Jenkinsfile`来定义管道。

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins12.png)

我这里配置了一个git仓库位置，然后我在该项目根目录放一个`Jenkinsfile`，其实就是我上一篇里演示的。

### Poll SCM 触发器

选择`Build Trigger`为`Poll SCM`，定时检查是否有push操作，这里我设置每隔2分钟检查一次。

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins13.png)

### Push触发器

这个触发器我更加推荐，因为是实时的，但是需要先配置gitlab的Webhook。

选择

```
Build when a change is pushed to GitLab. GitLab CI Service URL: http://192.168.217.161:8080/project/scm-example
```

复制后面那个URL，然后登录gitlab项目打开项目配置`Web Hook`，如果没有配置SSL可以将证书检查取消：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins14.png)

增加后可点击下面的`测试`，上面显示：`钩子执行成功：HTTP 200`则表示没问题。

然后再来jenkins里面配置push触发器，还能选择你要过滤那些分支，比如我只响应master分支上面的push操作。可以这样：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins15.png)

push钩子的大致流程是这样的：

1. push代码，Gitlab触发hook，访问Jenkins提供的api
2. `Jenkins Branch Filter`系统判断自己需要处理的分支是否有改动，如果有开始构建
3. 运行构建脚本

然后我再来测试下，修改代码，提交后push到远程仓库中，看到效果正常触发了：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins16.png)

Tips: 每个`Jenkinsfile`文件都应该以`#!groovy`为开头第一行

## 使用Jenkinsfile

接下来详细介绍一下怎样编写`Jenkinsfile`来完成各种复杂的任务。

Pipeline支持两种形式，一种是`Declarative`管道，一个是`Scripted`管道。

一个`Jenkinsfile`就是一个文本文件，里面定义了`Jenkins Pipeline`。
将这个文本文件放到项目的根目录下面，纳入版本系统。

### 部署三阶段

一般我们的持续交付都有三个部分：Build、Test、Deploy，典型写法：

```
// Declarative //
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Building..'
                sh 'make'
                archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true
            }
        }
        stage('Test') {
            steps {
                echo 'Testing..'
                /* `make check` returns non-zero on test failures,
                 *  using `true` to allow the Pipeline to continue nonetheless
                 */
                sh 'make check || true' ①
                junit '**/target/*.xml' ②
            }
        }
        stage('Deploy') {
            when {
                expression {
                    /*如果测试失败，状态为UNSTABLE*/
                    currentBuild.result == null || currentBuild.result == 'SUCCESS' ①
                }
            }
            steps {
                echo 'Deploying..'
                sh 'make publish'
            }
        }
    }
}
```

### 环境变量

Jenkins定了很多内置的环境变量，可在文档`localhost:8080/pipeline-syntax/globals#env`找到，
通过`env`直接使用它们：

```
echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}"
```

设置环境变量：

```
// Declarative //
pipeline {
    agent any
    environment {
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment {
                DEBUG_FLAGS = '-g'
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```

### 使用多个agent

```
// Declarative //
pipeline {
    agent none
    stages {
        stage('Build') {
            agent any
            steps {
                checkout scm
                sh 'make'
                stash includes: '**/target/*.jar', name: 'app' ①
            }
        }
        stage('Test on Linux') {
            agent { ②
                label 'linux'
            }
            steps {
                unstash 'app' ③
                sh 'make check'
            }
            post {
                always {
                    junit '**/target/*.xml'
                }
            }
        }
        stage('Test on Windows') {
            agent {
                label 'windows'
            }
            steps {
                unstash 'app'
                bat 'make check' ④
            }
            post {
                always {
                    junit '**/target/*.xml'
                }
            }
        }
    }
}
```

上面的例子，在任一台机器上面做Build操作，并通过`stash`命令保存文件，然后分别在两台agent机器上面做测试。
注意这里所有步骤都是串行执行的。

### Multibranch Pipeline

多分支管道可以让你在同一个项目中，对每个分支定义一个执行管道。Jenkins或自动发现、管理并执行包含`Jenkinsfile`文件的分支。

这个在前面一篇已经演示过怎样创建这样的Pipeline了，就不再多讲。

## Pipeline语法

先讲`Declarative Pipeline`，所有声明式管道都必须包含在`pipeline`块中：

```
pipeline {
    /* insert Declarative Pipeline here */
}
```

块里面的语句和表达式都是Groovy语法，遵循以下规则：

1. 最顶层规定就是`pipeline { }`
2. 语句结束不需要分好，一行一条语句
3. 块中只能包含`Sections`, `Directives`, `Steps`或者赋值语句
4. 属性引用语句被当成是无参方法调用，比如`input`实际上就是方法`input()`调用

接下来我详细讲解下`Sections`, `Directives`, `Steps`这三个东西

### Sections

`Sections`在声明式管道中包含一个或多个`Directives`, `Steps`

#### post

`post` section 定义了管道执行结束后要进行的操作。支持在里面定义很多`Conditions`块：
`always`, `changed`, `failure`, `success` 和 `unstable`。
这些条件块会根据不同的返回结果来执行不同的逻辑。

* always：不管返回什么状态都会执行
* changed：如果当前管道返回值和上一次已经完成的管道返回值不同时候执行
* failure：当前管道返回状态值为"failed"时候执行，在Web UI界面上面是红色的标志
* success：当前管道返回状态值为"success"时候执行，在Web UI界面上面是绿色的标志
* unstable：当前管道返回状态值为"unstable"时候执行，通常因为测试失败，代码不合法引起的。在Web UI界面上面是黄色的标志

```
// Declarative //
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
    post { ①
        always { ②
            echo 'I will always say Hello again!'
        }
    }
}
```

#### stages

由一个或多个`stage`指令组成，stages块也是核心逻辑的部分。
我们建议对于每个独立的交付部分（比如`Build`,`Test`,`Deploy`）都应该至少定义一个`stage`指令。比如：

```
// Declarative //
pipeline {
    agent any
    stages { ①
        stage('Example') {
        steps {
            echo 'Hello World'
        }
        }
    }
}
```

#### steps

在`stage`中定义一系列的`step`来执行命令。

```
// Declarative //
pipeline {
    agent any
    stages {
        stage('Example') {
            steps { ①
                echo 'Hello World'
            }
        }
    }
}
```

### Directives

jenkins中的各种指令

#### agent

`agent`指令指定整个管道或某个特定的`stage`的执行环境。它的参数可用使用：

1. any - 任意一个可用的agent
2. none - 如果放在pipeline顶层，那么每一个`stage`都需要定义自己的`agent`指令
3. label - 在jenkins环境中指定标签的agent上面执行，比如`agent { label 'my-defined-label' }`
4. node - `agent { node { label 'labelName' } }` 和 label一样，但是可用定义更多可选项
5. docker - 指定在docker容器中运行
6. dockerfile - 使用源码根目录下面的`Dockerfile`构建容器来运行

#### environment

`environment`定义键值对的环境变量

```
// Declarative //
pipeline {
    agent any
    environment { ①
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment { ②
                AN_ACCESS_KEY = credentials('my-prefined-secret-text') ③
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```

#### options

还能定义一些管道特定的选项，介绍几个常用的：

* skipDefaultCheckout - 在`agent`指令中忽略源码`checkout`这一步骤。
* timeout - 超时设置`options { timeout(time: 1, unit: 'HOURS') }`
* retry - 直到成功的重试次数`options { retry(3) }`
* timestamps - 控制台输出前面加时间戳`options { timestamps() }`

#### parameters

参数指令，触发这个管道需要用户指定的参数，然后在`step`中通过`params`对象访问这些参数。

```
// Declarative //
pipeline {
    agent any
    parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    }
    stages {
        stage('Example') {
            steps {
                echo "Hello ${params.PERSON}"
            }
        }
    }
}
```

#### triggers

触发器指令定义了这个管道何时该执行，一般我们会将管道和GitHub、GitLab、BitBucket关联，
然后使用它们的webhooks来触发，就不需要这个指令了。如果不适用`webhooks`，就可以定义两种`cron`和`pollSCM`

* cron - linux的cron格式`triggers { cron('H 4/* 0 0 1-5') }`
* pollSCM - jenkins的`poll scm`语法，比如`triggers { pollSCM('H 4/* 0 0 1-5') }`

```
// Declarative //
pipeline {
    agent any
    triggers {
        cron('H 4/* 0 0 1-5')
    }
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

#### stage

`stage`指令定义在`stages`块中，里面必须至少包含一个`steps`指令，一个可选的`agent`指令，以及其他stage相关指令。

```
// Declarative //
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

#### tools

定义自动安装并自动放入`PATH`里面的工具集合

```
// Declarative //
pipeline {
    agent any
    tools {
        maven 'apache-maven-3.0.1' ①
    }
    stages {
        stage('Example') {
            steps {
                sh 'mvn --version'
            }
        }
    }
}
```

注：① 工具名称必须预先在Jenkins中配置好了 → Global Tool Configuration.

#### 内置条件

* branch - 分支匹配才执行 `when { branch 'master' }`
* environment - 环境变量匹配才执行 `when { environment name: 'DEPLOY_TO', value: 'production' }`
* expression - groovy表达式为真才执行 `expression { return params.DEBUG_BUILD } }`

```
// Declarative //
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                branch 'production'
            }
            echo 'Deploying'
        }
    }
}
```

### Steps

这里就是实实在在的执行步骤了，每个步骤step都具体干些什么东西，
前面的`Sections`、`Directives`算控制逻辑和环境准备，这里的就是真实执行步骤。

这部分内容最多不可能全部讲完，[官方Step指南](https://jenkins.io/doc/pipeline/steps/) 包含所有的东西。

`Declared Pipeline`和`Scripted Pipeline`都能使用这些step，除了下面这个特殊的`script`。

一个特殊的step就是`script`，它可以让你在声明管道中执行脚本，使用groovy语法，这个非常有用：

```
// Declarative //
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
                script {
                    def browsers = ['chrome', 'firefox']
                    for (int i = 0; i < browsers.size(); ++i) {
                        echo "Testing the ${browsers[i]} browser"
                    }
                }
                script {
                    // 一个优雅的退出pipeline的方法，这里可执行任意逻辑
                    if( $VALUE1 == $VALUE2 ) {
                       currentBuild.result = 'SUCCESS'
                       return
                    }
                }
            }
        }
    }
}
```

最后列出来一个典型的`Scripted Pipeline`：

```
node('master') {
    checkout scm

    stage('Build') {
        docker.image('maven:3.3.3').inside {
            sh 'mvn --version'
        }
    }

    stage('Deploy') {
        if (env.BRANCH_NAME == 'master') {
            echo 'I only execute on the master branch'
        } else {
            echo 'I execute elsewhere'
        }
    }
}
```

可以看到，`Scripted Pipeline`没那么多东西，就是定义一个`node`，
里面多个`stage`，里面就是使用Groovy语法执行各个`step`了，非常简单和清晰，也非常灵活。

### 两种Pipeline比较

`Declarative Pipeline`相对简单，而且不需要学习groovy语法，对于日常的一般任务完全够用，
而`Scripted Pipeline`可通过Groovy语言的强大特性做任何你想做的事情。

## Blue Ocean

Jenkins最新整了个`Blue Ocean`出来，我觉得有必要用单独来介绍一下这个东西。

`Blue Ocean`重新设计了用户使用Jenkins的方式，给我们带来极大的方便，同时也兼容自由风格的任务定义。

### 安装

可以在当前Jenkins环境下面安装`Blue Ocean`插件，具体步骤：

1. 登录Jenkins服务器
2. 侧边栏点击"Manage Jenkins" -> "Manage Plugins"
3. 选择"Available"然后使用搜索框查找"Blue Ocean"
4. 在安装列点击checkbox
5. 选择"不重启安装"或"下载并重启后安装"

### 启动

安装好后会出现一个"Open Blue Ocean"的按钮，点击即可进入蓝色海洋：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins17.png)

界面如下：

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins18.png)

蓝色海洋果真是蓝色的，^_^

### Pipeline编辑器

使用管道编辑器是最简单的方式，可以来创建多个并行执行的任务。
编辑完保存后会自动保存为`Jenkinsfile`并放到源码管理系统中。

这里我演示一个在github中没有定义过`Jenkinsfile`的仓库，创建pipeline会默认写入`Jenkinsfile`文件。

首先需要在GitHub上面生成你的`Personal access tokens`，然后创建一个关联GitHub的管道，可视化编辑任务。

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins19.png)

编辑完`pipeline`后保存会执行一次commit和push操作

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins20.png)

个人感觉这种可视化简单点的倒可以，复杂的还是手动写吧。

弄好后你想执行某个pipeline，就先点下那个五角星，上面就会出现执行按钮了。

![](https://xnstatic-1253397658.file.myqcloud.com/jenkins21.png)

## FAQ

如果遇见`for (item : items)`报错`NotSerializableException`或者`Unserializable iterator`等等错误，
就将`foreach`循环改成传统C语言的循环：

```
for (int i = 0; i < cluster_nodes.size(); i++) {
    node = cluster_nodes[i]
}
```
