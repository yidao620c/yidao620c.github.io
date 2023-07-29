---
title: maven自动上传包和执行脚本
date: 2020-03-16 21:30:22 +0800
toc: true
categories: [ 开发工具 ]
tags: [ maven ]
---

本地开发联调的时候需要将代码快速更新至开发环境验证效果，无需走冗长的流水线发布流程，直接通过maven插件快速部署。

通过引入maven插件`maven-antrun-plugin`，可实现本地编译打包、scp复制到服务器、ssh远程执行脚本。
实现复制jar包到容器中，并最终重启容器的效果。

<!-- more -->

## 环境配置

第一步：在数据节点打开paas用户的sudo权限，通过visudo命令：

```
paas ALL=(root) NOPASSWD: ALL
```

第二步，复制`pom.xml`为`pom_dev.xml`，并在`.gitignore`文件中添加`pom_dev.xml`，将其排除在版本控制管理，
只在本地有用。然后在`pom_dev.xml`中引入`maven_antrun-plugin`插件，根据自己的环境配置修改里面的主机IP、命令脚本名。
每个服务有一个自己的脚本，放在固定的目录，脚本名称就是`构建目标名称.sh`。

下面的配置演示1个数据节点配置，如果是3节点，则可以复制多个`<execution>`节点，在每个节点中替换一下主机IP即可。

```xml

<plugin>
    <inherited>false</inherited>
    <groupId>org.apache.maven.plugin</groupId>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>3.0.0</version>
    <executions>
        <execution>
            <id>scp-ssh</id>
            <phase>package</phase>
            <goals>
                <goal>run</goal>
            </goals>
            <configuration>
                <target name="scp-ssh" description="copy to server">
                    <echo message="Remember to fill empty fields..."/>
                    <!-- file to be transferred-->
                    <scp trust="true" failonerror="true" verbose="off" sftp="true"
                         file="${project.build.directory}/${project.build.finalName}.jar"
                         todir="paas:Image0@Lalla123@10.10.10.10:/tmp/scripts/"/>
                    <sshexec trust="true" failonerror="true"
                             host="10.10.10.10" username="paas" password="Image0@Lalla123"
                             command="/tmp/scripts/${project.build.finalName}.sh" timeout="120000"/>
                    <taskdef name="scp" classname="org.apache.tools.ant.taskdefs.optional.ssh.Scp">
                        <classpath refid="maven.plugin.classpath"/>
                    </taskdef>
                </target>
            </configuration>
        </execution>
    </executions>
    <dependencies>
        <dependency>
            <groupId>org.apache.ant</groupId>
            <artifactId>ant-commons-net</artifactId>
            <version>1.10.8</version>
        </dependency>
        <dependency>
            <groupId>org.apache.ant</groupId>
            <artifactId>ant-jsch</artifactId>
            <version>1.10.8</version>
        </dependency>
    </dependencies>
</plugin>
```

第三步：编写脚本`/tmp/scripts/${project.build.finalName}.sh`，按照实际的部署逻辑编写，可在脚本中使用sudo命令。
参考后面的脚本示例。

第四步：关闭容器的监控健康检查，这样容器替换包的时候不会自动重启。

## 一键部署

上面环境准备好后，后面只需要在项目根目录执行下面的命令即可快速部署到环境中，一般在1分钟内。

```bash
mvn clean && mvn package -DskipTests=true -f pom_dev.xml
```

## 脚本示例

```bash
#!/bin/bash

cd /tmp/scripts/
#重命名
mv xx.jar app.jar
#修改属主
sudo chown paas:paas app.jar
# 找到服务对应的容器ID
docker_id=$(sudo docker ps | grep 服务名 | awk '{print $1}')
# 复制包到容器对应目录中替换原来的包
sudo docker cp app.jar ${docker_id}:/opt/data/
#删除临时包
rm -f app.jar
#重启容器
sudo docker restart ${docker_id}
```


