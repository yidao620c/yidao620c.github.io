---
title: Maven site发布多模块的项目站点
date: '2018-09-17 20:22:10 +0800'
toc: true
categories: [ 开发工具 ]
tags: [ maven ]
---

本地生成预览，修改父模块的pom.xml:

```xml

<site>
    <id>${project.artifactId}-site</id>
    <url>file://./</url>
</site>
```

执行

```bash
mvn clean && mvn site:site && mvn site:stage
```

目标站点在`target/stage`目录下面
<!-- more -->

## 部署到服务器

### 使用scp协议

如果使用scp协议，底层使用ssh协议，则需要配置操作系统用户认证

编辑maven的settings.xml文件，增加一个server配置

```xml

<server>
    <id>xx.xncoding.com</id>
    <username>name</username>
    <password>password</password>
</server>
```

修改父模块的pom.xml:

```xml

<plugin>
    <artifactId>maven-site-plugin</artifactId>
    <version>3.7.1</version>
    <dependencies>
        <dependency><!-- add support for ssh/scp -->
            <groupId>org.apache.maven.wagon</groupId>
            <artifactId>wagon-ssh</artifactId>
            <version>3.3.2</version>
        </dependency>
    </dependencies>
</plugin>

<site>
<id>xx.xncoding.com</id>
<url>scp://xx.xncoding.com/data/tomcat/webapps/xx/</url>
</site>
```

### 使用dav协议

首先需要对tomcat进行配置，开启WebDAV的支持。

1、在Tomcat的webapps目录下新建security文件夹，并在此文件夹下新建WEB-INF\web.xml文件。内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         id="WebApp_ID" version="3.0">
    <display-name>security sdk</display-name>
    <servlet>
        <servlet-name>webdav</servlet-name>
        <servlet-class>org.apache.catalina.servlets.WebdavServlet</servlet-class>
        <init-param>
            <param-name>debug</param-name>
            <param-value>0</param-value>
        </init-param>
        <init-param>
            <param-name>listings</param-name>
            <param-value>true</param-value>
        </init-param>
        <!-- Read-Write Access Settings -->
        <init-param>
            <param-name>readonly</param-name>
            <param-value>false</param-value>
        </init-param>
    </servlet>
    <!-- URL Mapping -->
    <servlet-mapping>
        <servlet-name>webdav</servlet-name>
        <url-pattern>/webdav/*</url-pattern>
    </servlet-mapping>
    <security-constraint>
        <web-resource-collection>
            <web-resource-name>webdav</web-resource-name>
            <!-- Detect WebDAV Methods in URL For Whole Application -->
            <url-pattern>/webdav/*</url-pattern>
            <http-method>PROPFIND</http-method>
            <http-method>PROPPATCH</http-method>
            <http-method>COPY</http-method>
            <http-method>MOVE</http-method>
            <http-method>LOCK</http-method>
            <http-method>UNLOCK</http-method>
        </web-resource-collection>
        <!-- Restrict access by role -->
        <auth-constraint>
            <role-name>*</role-name>
        </auth-constraint>
    </security-constraint>
    <login-config>
        <auth-method>BASIC</auth-method>
        <realm-name>webdav</realm-name>
    </login-config>
    <security-role>
        <description>WebDAV User</description>
        <role-name>webdav</role-name>
    </security-role>
</web-app>
```

根据上面权限名称，在Tomcat账号体系中增加账号密码，编辑/conf/tomcat-users.xml，内容如下：

```xml

<role rolename="tomcat"/>
<role rolename="admin-gui"/>
<role rolename="admin-script"/>
<role rolename="manager-gui"/>
<role rolename="webdav"/>
<user username="tomcat" password="tomcat" roles="tomcat,admin-gui,admin-script,manager-gui,webdav"/>
```

提示：权限名称必须和web.xml文件配置的一一对应。

2、然后跟上面一样，去本地配置maven的配置：

```xml

<server>
    <id>xx.xncoding.com</id>
    <username>tomcat</username>
    <password>tomcat</password>
</server>
```

然后maven的pom.xml配置：

```xml

<plugin>
    <artifactId>maven-site-plugin</artifactId>
    <version>3.7.1</version>
    <dependencies>
        <dependency>
            <groupId>org.apache.maven.wagon</groupId>
            <artifactId>wagon-webdav-jackrabbit</artifactId>
            <version>3.2.0</version>
        </dependency>
    </dependencies>
</plugin>

<site>
<id>xx.xncoding.com</id>
<url>dav:https://xx.xncoding.com/security/webdav/</url>
</site>
```

## 配置示例

主要是两个配置，一个是pom依赖：

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>xx.yy.xncoding.com</groupId>
    <artifactId>xx-sdk</artifactId>
    <packaging>pom</packaging>
    <version>1.0.0-SNAPSHOT</version>
    <modules>
        <module>xx-sdk-base</module>
        <module>xx-sdk-foo</module>
        <module>xx-sdk-bar</module>
    </modules>
    <name>xx-sdk</name>
    <description>description</description>
    <url>http://xx.xncoding.com/</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <servlet-api.version>3.1.0</servlet-api.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.slf4j</groupId>
                <artifactId>slf4j-api</artifactId>
                <version>1.7.25</version>
                <scope>provided</scope>
            </dependency>
            <dependency>
                <groupId>javax.servlet</groupId>
                <artifactId>javax.servlet-api</artifactId>
                <version>${servlet-api.version}</version>
                <scope>provided</scope>
            </dependency>
            <dependency>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
                <version>4.11</version>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <pluginManagement><!-- lock down plugins versions to avoid using Maven defaults (may be moved to parent pom) -->
            <plugins>
                <!-- clean lifecycle, see https://maven.apache.org/ref/current/maven-core/lifecycles.html#clean_Lifecycle -->
                <plugin>
                    <artifactId>maven-clean-plugin</artifactId>
                    <version>3.1.0</version>
                </plugin>
                <!-- default lifecycle, jar packaging: see https://maven.apache.org/ref/current/maven-core/default-bindings.html#Plugin_bindings_for_jar_packaging -->
                <plugin>
                    <artifactId>maven-resources-plugin</artifactId>
                    <version>3.0.2</version>
                </plugin>
                <plugin>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.8.0</version>
                </plugin>
                <plugin>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>2.22.1</version>
                </plugin>
                <plugin>
                    <artifactId>maven-jar-plugin</artifactId>
                    <version>3.0.2</version>
                </plugin>
                <plugin>
                    <artifactId>maven-source-plugin</artifactId>
                    <version>3.0.1</version>
                </plugin>
                <plugin>
                    <artifactId>maven-javadoc-plugin</artifactId>
                    <version>3.1.0</version>
                </plugin>
                <plugin>
                    <artifactId>maven-install-plugin</artifactId>
                    <version>2.5.2</version>
                </plugin>
                <plugin>
                    <artifactId>maven-deploy-plugin</artifactId>
                    <version>2.8.2</version>
                </plugin>
                <!-- site lifecycle, see https://maven.apache.org/ref/current/maven-core/lifecycles.html#site_Lifecycle -->
                <plugin>
                    <artifactId>maven-site-plugin</artifactId>
                    <version>3.7.1</version>
                    <dependencies>
                        <!--
                        <dependency><!-- add support for ssh/scp -->
                        <groupId>org.apache.maven.wagon</groupId>
                        <artifactId>wagon-ssh</artifactId>
                        <version>3.3.2</version>
                    </dependency>
                    -->
                    <dependency>
                        <groupId>org.apache.maven.wagon</groupId>
                        <artifactId>wagon-webdav-jackrabbit</artifactId>
                        <version>3.2.0</version>
                    </dependency>
                </dependencies>
            </plugin>
            <plugin>
                <artifactId>maven-project-info-reports-plugin</artifactId>
                <version>3.0.0</version>
            </plugin>
        </plugins>
    </pluginManagement>
</build>

<reporting>
<plugins>
    <plugin>
        <artifactId>maven-javadoc-plugin</artifactId>
        <configuration>
            <failOnError>false</failOnError>
        </configuration>
        <reportSets>
            <reportSet>
                <id>default</id>
                <reports>
                    <!--<report>javadoc</report>-->
                </reports>
                <configuration>
                    <aggregate>true</aggregate>
                </configuration>
            </reportSet>
            <reportSet><!-- aggregate reportSet, to define in poms having modules -->
                <id>aggregate</id>
                <inherited>false</inherited><!-- don't run aggregate in child modules -->
                <reports>
                    <report>aggregate</report>
                </reports>
            </reportSet>
        </reportSets>
    </plugin>
    <plugin>
        <artifactId>maven-project-info-reports-plugin</artifactId>
        <configuration>
            <customBundle>${project.basedir}/src/site/custom/project-info-reports.properties</customBundle>
        </configuration>
        <reportSets>
            <reportSet>
                <reports><!-- select reports -->
                    <report>index</report>
                    <report>summary</report>
                    <report>dependency-info</report>
                    <report>dependency-management</report>
                    <report>modules</report>
                    <report>plugin-management</report>
                    <report>team</report>
                </reports>
            </reportSet>
        </reportSets>
    </plugin>
</plugins>
</reporting>

<distributionManagement>
<snapshotRepository>
    <id>snapshot</id>
    <name>Snapshot</name>
    <url>http://xxxxxxx
    </url>
</snapshotRepository>

<!--<site>-->
<!--<id>${project.artifactId}-site</id>-->
<!--<url>file://./</url>-->
<!--</site>-->
<!--
<site>
    <id>xx.xncoding.com</id>
    <url>scp://xx.xncoding.com/data/tomcat/webapps/xx/</url>
</site>
-->
<site>
    <id>xx.xncoding.com</id>
    <url>dav:https://xx.xncoding.com/security/webdav/</url>
</site>
</distributionManagement>

<developers>
<developer>
    <id>xn</id>
    <name>XN</name>
    <email>xx</email>
    <url>http://www.xncoding.com</url>
    <organization>XX</organization>
    <organizationUrl>http://xx.com</organizationUrl>
    <roles>
        <role>architect</role>
        <role>developer</role>
    </roles>
</developer>
</developers>
        </project>
```

然后就是site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/DECORATION/1.8.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/DECORATION/1.8.0 http://maven.apache.org/xsd/decoration-1.8.0.xsd"
         name="Security SDK">

    <bannerLeft>
        <name>Project Name</name>
        <src>https://maven.apache.org/images/apache-maven-project.png</src>
        <href>https://maven.apache.org/</href>
    </bannerLeft>
    <bannerRight>
        <src>https://maven.apache.org/images/maven-logo-black-on-white.png</src>
    </bannerRight>
    <publishDate position="right"/>
    <version position="right"/>
    <poweredBy>
        <logo name="Maven" href="https://maven.apache.org/"
              img="https://maven.apache.org/images/logos/maven-feather.png"/>
    </poweredBy>

    <body>
        <head>
            <![CDATA[<link rel="shotcut icon" href="/security/favicon.ico"/>]]>
            <!-- Workaround for https://issues.apache.org/jira/browse/DOXIA-571 -->
            <![CDATA[<script type="text/javascript">
                $(document).ready(function () {
                    $(".source").addClass("prettyprint");
                    prettyPrint();
                });
            </script>]]>
        </head>
        <!--<links>-->
        <!--<item name="Apache" href="http://www.apache.org"/>-->
        <!--<item name="Maven" href="https://maven.apache.org"/>-->
        <!--</links>-->
        <breadcrumbs>
            <item name="Documentation" href="index.html"/>
            <item name="Examples" href="examples/auditlog.html"/>
        </breadcrumbs>
        <menu name="Overview">
            <item name="1 SDK简介" href="index.html" collapse="true">
            </item>
            <item name="2 集成指导" href="integrate.html" collapse="true">
            </item>
            <item name="3 运行日志" href="sdk/run-log/index.html" collapse="true">
                <item name="3.1 介绍" href="sdk/run-log/introduce.html" collapse="true"/>
                <item name="3.2 使用方法" href="sdk/run-log/usage.html" collapse="true"/>
            </item>
            <item name="4 审计日志" href="sdk/operation-log/index.html" collapse="true">
                <item name="4.1 介绍" href="sdk/operation-log/introduce.html" collapse="true"/>
                <item name="4.2 使用方法" href="sdk/operation-log/usage.html" collapse="true"/>
            </item>
            <item name="5 加密/解密" href="sdk/crypto/index.html" collapse="true">
                <item name="5.1 介绍" href="sdk/crypto/introduce.html" collapse="true"/>
                <item name="5.2 使用方法" href="sdk/crypto/usage.html" collapse="true"/>
            </item>
            <item name="9 工具集" href="sdk/tools/index.html" collapse="true">
                <item name="9.1 解压缩" href="sdk/tools/compress/index.html" collapse="true">
                    <item name="9.1.1 介绍" href="sdk/tools/compress/introduce.html" collapse="true"/>
                    <item name="9.1.2 使用方法" href="sdk/tools/compress/usage.html" collapse="true"/>
                </item>
                <item name="9.2 CSV校验" href="sdk/tools/csv/index.html" collapse="true">
                    <item name="9.2.1 介绍" href="sdk/tools/csv/introduce.html" collapse="true"/>
                    <item name="9.2.2 使用方法" href="sdk/tools/csv/usage.html" collapse="true"/>
                </item>
                <item name="9.3 HTTP工具" href="sdk/tools/http/index.html" collapse="true">
                    <item name="9.3.1 介绍" href="sdk/tools/http/introduce.html" collapse="true"/>
                    <item name="9.3.2 使用方法" href="sdk/tools/http/usage.html" collapse="true"/>
                </item>
                <item name="9.4 随机数" href="sdk/tools/random/index.html" collapse="true">
                    <item name="9.4.1 介绍" href="sdk/tools/random/introduce.html" collapse="true"/>
                    <item name="9.4.2 使用方法" href="sdk/tools/random/usage.html" collapse="true"/>
                </item>
                <item name="9.5 字符数组" href="sdk/tools/char-array/index.html" collapse="true">
                    <item name="9.5.1 介绍" href="sdk/tools/char-array/introduce.html" collapse="true"/>
                    <item name="9.5.2 使用方法" href="sdk/tools/char-array/usage.html" collapse="true"/>
                </item>
                <item name="9.6 字符串" href="sdk/tools/string/index.html" collapse="true">
                    <item name="9.6.1 介绍" href="sdk/tools/string/introduce.html" collapse="true"/>
                    <item name="9.6.2 使用方法" href="sdk/tools/string/usage.html" collapse="true"/>
                </item>
                <item name="9.7 XML" href="sdk/tools/xml/index.html" collapse="true">
                    <item name="9.7.1 介绍" href="sdk/tools/xml/introduce.html" collapse="true"/>
                    <item name="9.7.2 使用方法" href="sdk/tools/xml/usage.html" collapse="true"/>
                </item>
                <item name="9.8 URL" href="sdk/tools/url/index.html" collapse="true">
                    <item name="9.8.1 介绍" href="sdk/tools/url/introduce.html" collapse="true"/>
                    <item name="9.8.2 使用方法" href="sdk/tools/url/usage.html" collapse="true"/>
                </item>
                <item name="9.9 命令执行" href="sdk/tools/command/index.html" collapse="true">
                    <item name="9.9.1 介绍" href="sdk/tools/command/introduce.html" collapse="true"/>
                    <item name="9.9.2 使用方法" href="sdk/tools/command/usage.html" collapse="true"/>
                </item>
            </item>
            <item name="10 FAQ" href="faq.html" collapse="true">
            </item>
        </menu>
        <menu name="Documentation">
            <item name="JAVA DOC" href="apidocs/index.html" target="_blank"/>
        </menu>
        <!--<menu name="Examples">-->
        <!--<item name="上传操作日志" href="examples/auditlog.html"/>-->
        <!--<item name="获取客户端IP" href="examples/clientip.html"/>-->
        <!--<item name="执行OS命令" href="examples/command.html"/>-->
        <!--</menu>-->
        <menu name="PROJECT DOCUMENTATION">
            <item name="Project Infomation" href="project-info.html" collapse="true">
                <item name="Index" href="index.html"/>
                <item name="Summary" href="summary.html"/>
                <item name="Dependency Infomation" href="dependency-info.html"/>
                <item name="Dependency Management" href="dependency-management.html"/>
                <item name="Plugin Management" href="plugin-management.html"/>
                <item name="Team" href="team.html"/>
            </item>
        </menu>
        <!--<menu ref="reports"/>-->
        <!--<menu ref="modules"/>-->
    </body>

    <skin>
        <groupId>org.apache.maven.skins</groupId>
        <artifactId>maven-fluido-skin</artifactId>
        <version>1.7</version>
    </skin>
    <custom>
        <fluidoSkin>
            <sourceLineNumbersEnabled>true</sourceLineNumbersEnabled>
        </fluidoSkin>
    </custom>
</project>
```

自定义一些现实字段：`/src/site/custom/project-info-reports.properties`

## 源文档视图

```
src
└─site
    │  site.xml
    │
    ├─custom
    │      project-info-reports.properties
    │
    ├─markdown
    │  │  faq.md
    │  │  index.md
    │  │  integrate.md
    │  │  source.md
    │  │
    │  ├─examples
    │  │      auditlog.md
    │  │      clientip.md
    │  │      runlog.md
    │  │
    │  └─sdk
    │      ├─crypto
    │      │      index.md
    │      │      introduce.md
    │      │      usage.md
    │      │
    │      ├─operation-log
    │      │      index.md
    │      │      introduce.md
    │      │      usage.md
    │      │
    │      ├─run-log
    │      │      index.md
    │      │      introduce.md
    │      │      usage.md
    │      │
    │      └─tools
    │          │  index.md
    │          │
    │          ├─char-array
    │          │      index.md
    │          │      introduce.md
    │          │      usage.md
    │          │
    │          ├─command
    │          │      index.md
    │          │      introduce.md
    │          │      usage.md
    │          │
    │          ├─compress
    │          │      index.md
    │          │      introduce.md
    │          │      usage.md
    │          │
    │          ├─csv
    │          │      index.md
    │          │      introduce.md
    │          │      usage.md
    │          │
    │          ├─http
    │          │      index.md
    │          │      introduce.md
    │          │      usage.md
    │          │
    │          ├─random
    │          │      index.md
    │          │      introduce.md
    │          │      usage.md
    │          │
    │          ├─string
    │          │      index.md
    │          │      introduce.md
    │          │      usage.md
    │          │
    │          ├─url
    │          │      index.md
    │          │      introduce.md
    │          │      usage.md
    │          │
    │          └─xml
    │                  index.md
    │                  introduce.md
    │                  usage.md
    │
    └─resources
        │  favicon.ico
        │
        ├─css
        │      site.css
        │
        └─sdk
            ├─operation-log
            │      design01.png
            │
            └─tools
                └─char-array
                        design.png
```

每个文件夹中index.md示例：

```
## Summary

安全工具集合

* [字符数组检查](char-array/index.html)
* [CSV文件检查](csv/index.html)
* [HTTP工具](http/index.html)
* [IO工具](io/index.html)
* [随机数生成](random/index.html)
* [字符串操作工具](string/index.html)
* [文件解压缩](compress/index.html)
* [XML文档安全读取](xml/index.html)
```

## 执行命令

执行

```bash
mvn clean && mvn -Dmaven.wagon.http.ssl.insecure=true -Dmaven.wagon.http.ssl.allowall=true -Dmaven.wagon.http.ssl.ignore.validity.dates=true site-deploy
```

## 代码高亮和网站图标

在site.xml的body中添加如下片段，注意/security/是根据tomcat部署的子文件夹路径来的，如果部署在根路径下就是/favicon.ico即可:

```html

<body>
<head>
    <![CDATA[
    <link rel="shotcut icon" href="/security/favicon.ico"/>
    ]]>
    <!-- Workaround for https://issues.apache.org/jira/browse/DOXIA-571 -->
    <![CDATA[
    <script type="text/javascript">
        $(document).ready(function () {
            $(".source").addClass("prettyprint");
            prettyPrint();
        });
    </script>
    ]]>
</head>
</body>
```

## 自定义CSS

创建样式文件：src/site/resources/css/site.css，这个文件会被自动引入。内容如下：

```css
body {
    font-size: 15px;
}

h3 {
    font-size: 22.5px;
}

p {
    width: 70%;
}

.mytable {
    width: 70%;
    table-layout: fixed;
    border: 1px solid #3333336b;
}

.mytable th {
    background-color: rgb(210, 226, 255);
}

.mytable th, .mytable td {
    vertical-align: middle;
    word-wrap: break-word;
    word-break: break-all;
}

.thname {
    width: 15%
}

.thtype {
    width: 15%
}

.thmust {
    width: 8%
}

.thdesc {
    width: 40%
}

.thscope {
    width: 22%
}

pre.source {
    width: 70%;
    word-wrap: normal;
    white-space: pre;
    overflow-x: auto;
    display: inline-block;
}

pre.prettyprint {
    margin-bottom: 0;
}
```

自定义表格：

```html

<table class="table table-bordered table-condensed mytable" border="1">
    <tr>
        <th class="thname">参数名称</th>
        <th class="thtype">参数类型</th>
        <th class="thmust">是否必须</th>
        <th class="thdesc">参数描述</th>
        <th class="thscope">参数范围</th>
    </tr>
</table>
```

## markdown语法说明

参考：http://masikkk.com/article/MarkDown/

几个注意点:

1. 代码引用一定要使用反引号引起来
2. 如果是直接写html标签语法，比如在table中写代码，则那些特殊字符比如泛型的写法<>，请使用&lt;和&gt;来代替

