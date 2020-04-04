---
title: gitignore文件示例
date: '2018-10-06 21:22:18 +0800'
comments: true
toc: true
categories: fullstack
tags:
  - git
abbrlink: 25153
---

个人比较常用的.gitignore文件如下：
<!-- more -->

```
#idea
.idea/
target/
*.iml
*.ipr
*.iws
*.log

#eclipse
.project
.classpath
.classpath.log
pom.xml.versionsBackup
.settings

#gradle
.gradle
/build/
gradle-app.setting
.gradletasknamecache

#nodejs
node_modules/
package-lock.json

## others
.svn/
rebel.xml
.rebel-remote.xml.*
```

