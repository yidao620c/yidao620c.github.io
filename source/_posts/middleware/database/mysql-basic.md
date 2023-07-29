---
title: MySQL核心知识整理
date: 2016-09-17 12:07:14 +0800
toc: true
categories: [ 中间件 ]
tags: [ mysql ]
---

用了这么久的mysql数据库，是时候来一篇总结了，这里整理使用过程中觉得比较重要也比较常用的知识点。

开源数据库里面mysql的份额最高，也是最流行的，其实性能也非常高，所以一直用它。

在CentOS 7.2里面通过yum来安装MySQL 5.7，同时配置好root密码以及允许其他ip访问。

从 MySQL 官网选取合适的 MySQL 版本，获取下载地址。然后使用 `wget` 下载：

```bash
wget http://repo.mysql.com/mysql57-community-release-el7-8.noarch.rpm
```

安装 yum Repository

```bash
yum -y install mysql57-community-release-el7-8.noarch.rpm
```

<!-- more -->

搜索 mysql server

```bash
yum search mysql-com
```

安装

```bash
yum -y install mysql-community-server.x86_64
```

启动、停止、查看状态、开机启动等

```bash
systemctl start mysqld.service
systemctl stop mysqld.service
systemctl status mysqld.service
systemctl enable mysqld.service
```

登陆数据库

MySQL5.7.6 之后会在启动 mysql 进程的时候生成一个用户密码，首次登陆需要这个密码才行。
密码保存在 mysql 进程的日志里，即`/var/log/mysqld.log`

```bash
cat /var/log/mysqld.log | grep 'password'
# 登录
mysql -uroot -p
```

修改root密码，先登录进去后：

```
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password';
```

最后，安装完了可以删除 MySQL 的 Repository ，这样可以减少 yum 检查更新的时间，使用下面的命令：

```bash
yum -y remove mysql57-community-release-el7-8.noarch
```

修改权限，让其他的机器也能访问：

```
mysql> GRANT ALL ON *.* TO 'root'@'%' IDENTIFIED BY 'mysql';
mysql> flush privileges;
```

如果还不能远程访问，就需要关闭防火墙。

## 常见数据库操作

创建新的数据库并制定UTF-8编码

```
drop database fastloan3;
create database fastloan3 DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
```

通过sql脚本文件执行sql：

```bash
mysql -uroot -p... fastloan3 < t_invoice.sql
```

导出数据库

```bash
mysqldump -u... -p... mydb > mydb_tables.sql
mysqldump -u... -p... mydb t1 t2 t3 > mydb_tables.sqlhe
```

查看表空间占用大小

```
mysql> use information_schema;
mysql> SELECT TABLE_NAME,DATA_LENGTH+INDEX_LENGTH,TABLE_ROWS
FROM TABLES WHERE TABLE_SCHEMA='数据库名' //AND TABLE_NAME='表名'
```

查看详细表结构，包括注释

```
show full columns from <table_name>
SELECT COLUMN_NAME, COLUMN_TYPE, EXTRA, COLUMN_COMMENT
FROM information_schema.columns
WHERE table_name = 'metrics' AND table_schema = 'fastloan3';
```

## 关于表连接

SQL的Join语法有很多inner的，有outer的，有left的，有时候对于Select出来的结果集是什么样子有点不是很清楚。
这里我通过实际例子来演示它们的用法。

假设我们有两张表。TableA是左边的表。TableB是右边的表。其各有5条记录，
其中有4条记录name是相同的，如下所示：让我们看看不同JOIN的不同

**A表**

 id | name   
----|--------
 1  | HaHa   
 2  | HeiHei 
 3  | WaWa   
 4  | SaSa   
 5  | YaYa   

**B表**

 id | name   
----|--------
 1  | HeiHei 
 2  | WaWa   
 3  | SaSa   
 4  | YaYa   
 5  | ZaZa   

### INNER JOIN

`INNER JOIN`跟`JOIN`是一样的，一般`INNER`关键字可以省略。
`INNER JOIN`将只会返回相匹配的元素项，即不会返回结果为NULL的数据项。

```sql
SELECT *
FROM TableA
         INNER JOIN TableB ON TableA.name = TableB.name
```

结果集

 id | name   | id1 | name1  
----|--------|-----|--------
 2  | HeiHei | 1	  | HeiHei 
 3  | WaWa   | 2	  | WaWa   
 4  | SaSa   | 3	  | SaSa   
 5  | YaYa   | 4	  | YaYa   

### LEFT [OUTER] JOIN

请注意这里的`OUTER`是可选的，`LEFT JOIN` 和 `LEFT OUTER JOIN`是一样的。
左连接（`LEFT OUTER JOIN`）会输出左表中的所有结果，如果右表中有相应项，则会输出，否则为NULL。

```sql
SELECT *
FROM TableA
         LEFT OUTER JOIN TableB ON TableA.name = TableB.name
```

结果集

 id | name   | id1  | name1  
----|--------|------|--------
 2  | HeiHei | 1	   | HeiHei 
 3  | WaWa   | 2	   | WaWa   
 4  | SaSa   | 3	   | SaSa   
 5  | YaYa   | 4	   | YaYa   
 1  | HaHa   | NULL | NULL   

### RIGHT [OUTER] JOIN

右连接（`RIGHT OUTER JOIN`）会输出右表中的所有结果，如果左表中有相应项，则会输出，否则为NULL。

```sql
SELECT *
FROM TableA
         RIGHT OUTER JOIN TableB ON TableA.name = TableB.name
```

结果集

 id   | name   | id1 | name1  
------|--------|-----|--------
 2    | HeiHei | 1   | HeiHei 
 3    | WaWa   | 2   | WaWa   
 4    | SaSa   | 3   | SaSa   
 5    | YaYa   | 4   | YaYa   
 NULL | NULL   | 5   | ZaZa   

### UNION 与 UNION ALL

`UNION`操作符用于合并两个或多个 `SELECT` 语句的结果集。请注意，`UNION` 内部的 `SELECT` 语句必须拥有相同数量的列。
列也必须拥有相似的数据类型。同时，每条`SELECT`语句中的列的顺序必须相同。`UNION`只选取不同的记录，而`UNION ALL`会列出所有记录。

```sql
SELECT name
FROM TableA
UNION
SELECT name
FROM TableB
```

结果集（注意下面没有重复项）

```
HaHa
HeiHei
WaWa
SaSa
YaYa
ZaZa
```

使用`UNION ALL`

```sql
SELECT name
FROM TableA
UNION ALL
SELECT name
FROM TableB
```

结果集

```
HaHa
HeiHei
WaWa
SaSa
YaYa
HeiHei
WaWa
SaSa
YaYa
ZaZa
```

## 分组查询GROUP BY

很多时候需要分组统计查询，这里就需要使用到`GROUP BY`，以及`HAVING`子句。

在 SQL 中增加 HAVING 子句原因是，WHERE 关键字无法与合计函数一起使用。

```sql
SELECT column_name, aggregate_function(column_name)
FROM table_name
WHERE column_name operator value
GROUP BY column_name
HAVING aggregate_function(column_name) operator value
```

比如我们拥有下面一个`Orders`表：

 O_Id | OrderDate  | OrderPrice | Customer 
------|------------|------------|----------
 1    | 2008/12/29 | 1000       | Bush     
 2    | 2008/11/23 | 1600       | Carter   
 3    | 2008/10/05 | 700        | Bush     
 4    | 2008/09/28 | 300        | Bush     
 5    | 2008/08/06 | 2000       | Adams    
 6    | 2008/07/21 | 100        | Carter   

现在，我们希望查找订单总金额少于 2000 的客户:

```sql
SELECT Customer, SUM(OrderPrice)
FROM Orders
GROUP BY Customer
HAVING SUM(OrderPrice) < 2000
```

现在我们希望查找客户 "Bush" 或 "Adams" 拥有超过 1500 的订单总金额。
我们在 SQL 语句中增加了一个普通的 WHERE 子句：

```sql
SELECT Customer, SUM(OrderPrice)
FROM Orders
WHERE Customer = 'Bush'
   OR Customer = 'Adams'
GROUP BY Customer
HAVING SUM(OrderPrice) > 1500
```

结果集：

 Customer | SUM(OrderPrice) 
----------|-----------------
 Bush     | 2000            
 Adams    | 2000            

现在我要求"Bush" 或 "Adams" 拥有超过 1500 的订单平均金额：

```sql
SELECT Customer, SUM(OrderPrice)
FROM Orders
WHERE Customer = 'Bush'
   OR Customer = 'Adams'
GROUP BY Customer
HAVING AVG(OrderPrice) > 1500
```

结果集：

 Customer | SUM(OrderPrice) 
----------|-----------------
 Adams    | 2000            

Bush被淘汰了，因为他的平均订单金额（(1000+700+300)/3）<1500

## 其他有用函数

`MID(str,pos,len)`提取子串，pos从1开始。

```sql
SELECT MID(Customer, 1, 3)
FROM Orders;
```
