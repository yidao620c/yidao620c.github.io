---
title: 使用smart-doc生成API文档
date: 2021-03-07 09:17:12 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

smart-doc是一款同时支持JAVA REST API和Apache Dubbo RPC接口文档生成的工具，smart-doc在业内率先提出基于JAVA泛型定义推导的理念，
完全基于接口源码来分析生成接口文档，不采用任何注解侵入到业务代码中。你只需要按照java-doc标准编写注释，smart-doc就能帮你生成一个简易明了的Markdown、HTML5、Postman
Collection2.0+、OpenAPI 3.0+的文档。

官方地址：https://gitee.com/smart-doc-team/smart-doc

<!-- more -->

## 支持的特性

* 零注解、零学习成本、只需要写标准JAVA注释。
* 基于源代码接口定义自动推导，强大的返回结构推导。
* 支持Spring MVC、Spring Boot、Spring Boot Web Flux(controller书写方式)、Feign。
* 支持Callable、Future、CompletableFuture等异步接口返回的推导。
* 支持JavaBean上的JSR303参数校验规范，包括分组验证。
* 对JSON请求参数的接口能够自动生成模拟JSON参数。
* 对一些常用字段定义能够生成有效的模拟值。
* 支持生成JSON返回值示例。
* 支持从项目外部加载源代码来生成字段注释(包括标准规范发布的jar包)。
* 支持生成多种格式文档：Markdown、HTML5、Asciidoctor、Postman Collection、OpenAPI 3.0。
* 轻易实现在Spring Boot服务上在线查看静态HTML5 api文档，支持生成在线debug页面。
* 开放文档数据，可自由实现接入文档管理系统。
* 支持导出错误码和定义在代码中的各种字典码到接口文档。
* 支持Maven、Gradle插件式轻松集成。
* 支持Apache Dubbo RPC接口文档生成。
* debug接口调试html5页面完全支持文件上传，下载(@download tag标记下载方法)测试。

## 使用方法

smart-doc官方目前已经开发完成Maven插件 和Gradle插件，你可以根据自己的构建工具来选择使用Maven插件或者是Gradle插件。

```xml

<plugin>
    <groupId>com.github.shalousun</groupId>
    <artifactId>smart-doc-maven-plugin</artifactId>
    <version>[最新版本]</version>
    <configuration>
        <!--指定生成文档的使用的配置文件,配置文件放在自己的项目中-->
        <configFile>./src/main/resources/smart-doc.json</configFile>
        <!--指定项目名称-->
        <projectName>测试</projectName>
        <!--smart-doc实现自动分析依赖树加载第三方依赖的源码，如果一些框架依赖库加载不到导致报错，这时请使用excludes排除掉-->
        <excludes>
            <!--格式为：groupId:artifactId;参考如下-->
            <exclude>com.alibaba:fastjson</exclude>
        </excludes>
        <!--自1.0.8版本开始，插件提供includes支持-->
        <!--smart-doc能自动分析依赖树加载所有依赖源码，原则上会影响文档构建效率，因此你可以使用includes来让插件加载你配置的组件-->
        <includes>
            <!--格式为：groupId:artifactId;参考如下-->
            <include>com.alibaba:fastjson</include>
        </includes>
    </configuration>
    <executions>
        <execution>
            <!--如果不需要在执行编译时启动smart-doc，则将phase注释掉-->
            <phase>compile</phase>
            <goals>
                <!--smart-doc提供了html、openapi、markdown等goal，可按需配置-->
                <goal>html</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

在项目中添加创建一个smart-doc.json配置文件，插件读取这个配置来生成项目的文档， 这个配置内容实际上就是以前采用单元测试编写的ApiConfig转成json后的结果，因此关于配置项说明可以参考原来单元测试的配置。 最小配置如下：

```json
{
  "outPath": "D://md2"
}
```

仅仅需要上面一行配置就能启动smart-doc-maven-plugin插件，根据自己项目情况更多详细的配置参考下面。 详细配置说明

```
{
  "serverUrl": "http://127.0.0.1", //服务器地址,非必须。导出postman建议设置成http://{{server}}方便直接在postman直接设置环境变量
  "isStrict": false, //是否开启严格模式
  "allInOne": true,  //是否将文档合并到一个文件中，一般推荐为true
  "outPath": "D://md2", //指定文档的输出路径
  "coverOld": true,  //是否覆盖旧的文件，主要用于mardown文件覆盖
  "createDebugPage": true,//@since 2.0.0 smart-doc支持创建可以测试的html页面，仅在AllInOne模式中起作用。
  "packageFilters": "",//controller包过滤，多个包用英文逗号隔开
  "md5EncryptedHtmlName": false,//只有每个controller生成一个html文件是才使用
  "style":"xt256", //基于highlight.js的代码高设置,可选值很多可查看码云wiki，喜欢配色统一简洁的同学可以不设置
  "projectName": "smart-doc",//配置自己的项目名称
  "skipTransientField": true,//目前未实现
  "sortByTitle":false,//接口标题排序，默认为false,@since 1.8.7版本开始
  "showAuthor":true,//是否显示接口作者名称，默认是true,不想显示可关闭
  "requestFieldToUnderline":true,//自动将驼峰入参字段在文档中转为下划线格式,//@since 1.8.7版本开始
  "responseFieldToUnderline":true,//自动将驼峰入参字段在文档中转为下划线格式,//@since 1.8.7版本开始
  "inlineEnum":true,//设置为true会将枚举详情展示到参数表中，默认关闭，//@since 1.8.8版本开始
  "recursionLimit":7,//设置允许递归执行的次数用于避免一些对象解析卡主，默认是7，正常为3次以内，//@since 1.8.8版本开始
  "allInOneDocFileName":"index.html",//自定义设置输出文档名称, @since 1.9.0
  "requestExample":"true",//是否将请求示例展示在文档中，默认true，@since 1.9.0
  "responseExample":"true",//是否将响应示例展示在文档中，默认为true，@since 1.9.0
  "urlSuffix":".do",//支持SpringMVC旧项目的url后缀,@since 2.1.0
  "displayActualType":false,//配置true会在注释栏自动显示泛型的真实类型短类名，@since 1.9.6
  "appKey": "20201216788835306945118208",// torna平台对接appKey,, @since 2.0.9
  "appToken": "c16931fa6590483fb7a4e85340fcbfef", //torna平台appToken,@since 2.0.9
  "secret": "W.ZyGMOB9Q0UqujVxnfi@.I#V&tUUYZR",//torna平台secret，@since 2.0.9
  "openUrl": "http://localhost:7700/api",//torna平台地址，填写自己的私有化部署地址@since 2.0.9
  "debugEnvName":"测试环境", //torna测试环境
  "debugEnvUrl":"http://127.0.0.1",//torna,和serverUrl会有冲突，设置serverUrl则可以不设置该选项
  "ignoreRequestParams":[ //忽略请求参数对象，把不想生成文档的参数对象屏蔽掉，@since 1.9.2
     "org.springframework.ui.ModelMap"
   ],
  "dataDictionaries": [{ //配置数据字典，没有需求可以不设置
      "title": "http状态码字典", //数据字典的名称
      "enumClassName": "com.power.common.enums.HttpCodeEnum", //数据字典枚举类名称
      "codeField": "code",//数据字典字典码对应的字段名称
      "descField": "message"//数据字典对象的描述信息字典
  }],
  "errorCodeDictionaries": [{ //错误码列表，没有需求可以不设置
    "title": "title",
    "enumClassName": "com.power.common.enums.HttpCodeEnum", //错误码枚举类
    "codeField": "code",//错误码的code码字段名称
    "descField": "message"//错误码的描述信息对应的字段名
  }],
  "revisionLogs": [{ //文档变更记录
      "version": "1.0", //文档版本号
      "revisionTime": "2020-12-31 10:30", //文档修订时间
      "status": "update", //变更操作状态，一般为：创建、更新等
      "author": "author", //文档变更作者
      "remarks": "desc" //变更描述
    }
  ],
  "customResponseFields": [{ //自定义添加字段和注释，api-doc后期遇到同名字段则直接给相应字段加注释，非必须
      "name": "code",//覆盖响应码字段
      "desc": "响应代码",//覆盖响应码的字段注释
      "ownerClassName": "org.springframework.data.domain.Pageable", //指定你要添加注释的类名
      "value": "00000"//设置响应码的值
  }],
  "requestHeaders": [{ //设置请求头，没有需求可以不设置
      "name": "token",//请求头名称
      "type": "string",//请求头类型
      "desc": "desc",//请求头描述信息
      "value":"token请求头的值",//不设置默认null
      "required": false,//是否必须
      "since": "-"//什么版本添加的改请求头
  }],
  "rpcApiDependencies":[{ // 项目开放的dubbo api接口模块依赖，配置后输出到文档方便使用者集成
        "artifactId":"SpringBoot2-Dubbo-Api",
        "groupId":"com.demo",
        "version":"1.0.0"
   }],
  "rpcConsumerConfig": "src/main/resources/consumer-example.conf",//文档中添加dubbo consumer集成配置，用于方便集成方可以快速集成
  "apiObjectReplacements": [{ // 自smart-doc 1.8.5开始你可以使用自定义类覆盖其他类做文档渲染，非必须
      "className": "org.springframework.data.domain.Pageable",
      "replacementClassName": "com.power.doc.model.PageRequestDto" //自定义的PageRequestDto替换Pageable做文档渲染
  }],
  "apiConstants": [{//从1.8.9开始配置自己的常量类，smart-doc在解析到常量时自动替换为具体的值
        "constantsClassName": "com.power.doc.constants.RequestParamConstant"
  }],
  "responseBodyAdvice":{ //自smart-doc 1.9.8起，ResponseBodyAdvice统一返回设置，可用ignoreResponseBodyAdvice tag来忽略
  		"className":"com.power.common.model.CommonResult" //通用响应体
  },
  "sourceCodePaths": [{ //设置代码路径, 插件已经能够自动下载发布的源码包，没必要配置
      "path": "src/main/java",
      "desc": "测试"
  }]
}
```

上面的JSON配置实例中只有"outPath"是必填项。

添加好插件和配置文件后可以直接运行Maven命令生成文档。

```
//生成html
mvn -Dfile.encoding=UTF-8 smart-doc:html
//生成markdown
mvn -Dfile.encoding=UTF-8 smart-doc:markdown
//生成adoc
mvn -Dfile.encoding=UTF-8 smart-doc:adoc
//生成postman json数据
mvn -Dfile.encoding=UTF-8 smart-doc:postman
// 生成 Open Api 3.0+,Since smart-doc-maven-plugin 1.1.5
mvn -Dfile.encoding=UTF-8 smart-doc:openapi
// 生成文档推送到Torna平台
mvn -Dfile.encoding=UTF-8 smart-doc:torna-rest

// Apache Dubbo RPC文档
// Generate html
mvn -Dfile.encoding=UTF-8 smart-doc:rpc-html
// Generate markdown
mvn -Dfile.encoding=UTF-8 smart-doc:rpc-markdown
// Generate adoc
mvn -Dfile.encoding=UTF-8 smart-doc:rpc-adoc
```

注意：尤其在window系统下，如果实际使用Maven命令行执行文档生成，可能会出现乱码，因此需要在执行时指定`-Dfile.encoding=UTF-8`。

## Markdown转换为Word

经常有需要将Markdown转换成Word、PDF、HTML等其他格式。这里使用pandoc轻松搞定这一切。

首先下载pandoc，地址为<https://pandoc.org/installing.html>
，windows版本下载msi安装文件直接安装即可。然后将pandoc可执行文件设置到PATH中去，如果选择的是.msi这个版本会自动配置环境变量。

然后执行：

```bash
pandoc -s source.md -o target.docx
```


