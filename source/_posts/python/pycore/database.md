---
title: python核心 - 访问数据库
date: 2015-12-24 22:22:22 +0800
toc: true
categories: [ python ]
tags: [ python核心 ]
---

操作数据库是最常见的任务，这里用MySQL来做演示，也是我们用的最多的一个开源数据库，其他都类似的。

对于安装MySQL就不做介绍了，安装完后，还需要安装去驱动。因为需要支持Python的MySQL驱动来连接到MySQL服务器。
MySQL的驱动有多种实现，比如纯python实现的pymysql和mysql-connector，或者mysql-python也就是MySQLdb。

这里我通过mysql-connector来介绍使用方法：

```bash
pip install mysql-connector
```

<!-- more -->

我们演示如何连接到MySQL服务器的test数据库：

```
# 导入MySQL驱动:
>>> import mysql.connector
# 注意把password设为你的root口令:
>>> conn = mysql.connector.connect(user='root', password='password', database='test')
>>> cursor = conn.cursor()
# 创建user表:
>>> cursor.execute('create table user (id varchar(20) primary key, name varchar(20))')
# 插入一行记录，注意MySQL的占位符是%s:
>>> cursor.execute('insert into user (id, name) values (%s, %s)', ['1', 'Michael'])
>>> cursor.rowcount
1
# 提交事务:
>>> conn.commit()
>>> cursor.close()
# 运行查询:
>>> cursor = conn.cursor()
>>> cursor.execute('select * from user where id = %s', ('1',))
>>> values = cursor.fetchall()
>>> values
[('1', 'Michael')]
# 关闭Cursor和Connection:
>>> cursor.close()
True
>>> conn.close()
```

由于Python的DB-API定义都是通用的，所以，操作MySQL的数据库代码和SQLite类似。
执行INSERT等操作后要调用commit()提交事务，MySQL的SQL占位符是%s。

## 使用SQLAlchemy

为了更加方便的操作数据库，我们通常会用到ORM框架，在python里面最著名的就是SQLAlchemy了，
关于这个我写了一篇专门的文章，叫"SQLAlchemy入门"，可以去看看，这里我就不多说了。





