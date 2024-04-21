---
title: 数据库文档生成工具screw
date: '2020-03-12 20:20:05 +0800'
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

在企业级开发中、我们经常会有编写数据库表结构文档的时间付出，如果数据库表结构更新了还得手动更新维护到文档中，很是繁琐。
无意之间发现了github上面有个人写了一个小工具专门来做这个事情，名字叫screw（螺丝刀），用了下很不错。这里特意记录一下。

工具的github地址：<https://github.com/pingfangushi/screw>

<!--more-->

## 特点

* 简洁、轻量、设计良好
* 多数据库支持，目前已支持MySQL、MariaDB、TIDB、Oracle、SqlServer、PostgreSQL、Cache DB
* 多种格式文档，目前已至此HTML、Word、MarkDown格式
* 灵活扩展
* 支持自定义模板

## 使用

有两种使用方式，一种是通过maven插件引入后执行命令生成，一种是直接写代码来生成文档。
下面我通过MySQL数据库的使用例子来说明。

### 引入依赖

```xml

<dependencies>
    <!-- screw核心 -->
    <dependency>
        <groupId>cn.smallbun.screw</groupId>
        <artifactId>screw-core</artifactId>
        <version>1.0.5</version>
    </dependency>
    <!-- HikariCP -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>3.4.5</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.20</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 代码方式

```java
public class BuildDatabaseDoc {
    /**
     * 文档生成
     */
    public static void main(String[] args) {
       //数据源
       HikariConfig hikariConfig = new HikariConfig();
       hikariConfig.setDriverClassName("com.mysql.cj.jdbc.Driver");
       hikariConfig.setJdbcUrl("jdbc:mysql://localhost:3306/oauth2?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=UTC");
       hikariConfig.setUsername("user");
       hikariConfig.setPassword("password");
       //设置可以获取tables remarks信息
       hikariConfig.addDataSourceProperty("useInformationSchema", "true");
       hikariConfig.setMinimumIdle(2);
       hikariConfig.setMaximumPoolSize(5);
       DataSource dataSource = new HikariDataSource(hikariConfig);
       //生成配置
       EngineConfig engineConfig = EngineConfig.builder()
             //生成文件路径
             .fileOutputDir(fileOutputDir)
             //打开目录
             .openOutputDir(true)
             //文件类型
             .fileType(EngineFileType.HTML)
             //生成模板实现
             .produceType(EngineTemplateType.freemarker)
             //自定义文件名称
             .fileName("自定义文件名称").build();
    
       //忽略表
       ArrayList<String> ignoreTableName = new ArrayList<>();
       ignoreTableName.add("test_user");
       ignoreTableName.add("test_group");
       //忽略表前缀
       ArrayList<String> ignorePrefix = new ArrayList<>();
       ignorePrefix.add("test_");
       //忽略表后缀    
       ArrayList<String> ignoreSuffix = new ArrayList<>();
       ignoreSuffix.add("_test");
       ProcessConfig processConfig = ProcessConfig.builder()
             //指定生成逻辑、当存在指定表、指定表前缀、指定表后缀时，将生成指定表，其余表不生成、并跳过忽略表配置	
             //根据名称指定表生成
             .designatedTableName(new ArrayList<>())
             //根据表前缀生成
             .designatedTablePrefix(new ArrayList<>())
             //根据表后缀生成	
             .designatedTableSuffix(new ArrayList<>())
             //忽略表名
             .ignoreTableName(ignoreTableName)
             //忽略表前缀
             .ignoreTablePrefix(ignorePrefix)
             //忽略表后缀
             .ignoreTableSuffix(ignoreSuffix).build();
       //配置
       Configuration config = Configuration.builder()
             //版本
             .version("1.0.0")
             //描述
             .description("数据库设计文档生成")
             //数据源
             .dataSource(dataSource)
             //生成配置
             .engineConfig(engineConfig)
             //生成配置
             .produceConfig(processConfig)
             .build();
       //执行生成
       new DocumentationExecute(config).execute();
    }
}
```

### maven插件方式

```xml

<build>
    <plugins>
        <plugin>
            <groupId>cn.smallbun.screw</groupId>
            <artifactId>screw-maven-plugin</artifactId>
            <version>${lastVersion}</version>
            <dependencies>
                <!-- HikariCP -->
                <dependency>
                    <groupId>com.zaxxer</groupId>
                    <artifactId>HikariCP</artifactId>
                    <version>3.4.5</version>
                </dependency>
                <!--mysql driver-->
                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>8.0.20</version>
                </dependency>
            </dependencies>
            <configuration>
                <!--username-->
                <username>root</username>
                <!--password-->
                <password>password</password>
                <!--driver-->
                <driverClassName>com.mysql.cj.jdbc.Driver</driverClassName>
                <!--jdbc url-->
                <jdbcUrl>jdbc:mysql://127.0.0.1:3306/xxxx</jdbcUrl>
                <!--生成文件类型-->
                <fileType>HTML</fileType>
                <!--打开文件输出目录-->
                <openOutputDir>false</openOutputDir>
                <!--生成模板-->
                <produceType>freemarker</produceType>
                <!--文档名称 为空时:将采用[数据库名称-描述-版本号]作为文档名称-->
                <fileName>测试文档名称</fileName>
                <!--描述-->
                <description>数据库文档生成</description>
                <!--版本-->
                <version>${project.version}</version>
                <!--标题-->
                <title>数据库文档</title>
            </configuration>
            <executions>
                <execution>
                    <phase>compile</phase>
                    <goals>
                        <goal>run</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

配置完以后在 `maven project` -> `screw` -> `screw:run` 双击执行ok。

### 效果图

![](https://xnstatic-1253397658.file.myqcloud.com/dbdocs.png)

