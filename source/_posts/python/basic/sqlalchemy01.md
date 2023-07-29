---
title: SQLAlchemy入门
date: 2016-03-07 10:12:42 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

SQLAlchemy是Python世界中最广泛使用的ORM工具之一，它采用了类似于Java里Hibernate的数据映射模型，
而不是其他ORM框架采用的`Active Record`模型。

SQLAlchemy分为两个部分，一个是最常用的ORM对象映射，另一个是核心的`SQL expression`。
第一个很好理解，纯粹的ORM，后面这个不是ORM，而是DBAPI的封装，通过一些sql表达式来避免了直接写sql。

使用`SQLAlchemy`则可以分为三种方式。

* 使用ORM避免直接书写sql
* 使用raw sql直接书写sql
* 使用sql expression，通过SQLAlchemy的方法写sql表达式

<!-- more -->

### 安装

最简单的方式是通过pip安装

```bash
pip install SQLAlchemy
```

一般来讲我们要对某个底层数据库需要安装相应的驱动，比如我使用了mysql，那么需要安装python的mysql驱动，有很多种选择，
这里我选择了MySQLdb/MySQL-Python，这也是SQLAlchemy默认的。

在centos上面安装MySQL-Python

```bash
yum install mysql-devel
pip install MySQL-python
```

注意：MySQLdb仅仅支持python2，如果要支持python3，安装PyMySQL：

```bash
pip install PyMySQL
```

这里我使用python3.6版本来测试

### 定义映射

这里我使用两个表来说明，一个用户表users，一个电子邮件表addresses，两者一对多的关系。我们先定义这两个映射：

```python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

Base = declarative_base()


class Address(Base):
    """电子邮件表"""
    __tablename__ = 'addresses'

    id = Column(Integer, primary_key=True)
    email_address = Column(String(30), nullable=False)
    user_id = Column(Integer, ForeignKey('users.id'))
    user = relationship("User", back_populates="addresses")

    def __repr__(self):
        return "<Address(email_address='{}')>".format(self.email_address)


class User(Base):
    """用户表"""
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String(10))
    fullname = Column(String(20))
    password = Column(String(20))

    addresses = relationship("Address", order_by=Address.id, back_populates="user")

    def __repr__(self):
        return "<User(name='{}', fullname='{}', password='{}')>".format(
            self.name, self.fullname, self.password)


```

### 连接到数据库

通过`create_engine()`可以连接数据库，我使用的是PyMySQL，另外先要提前创建test这个测试数据库：

```python
from sqlalchemy import create_engine

# 下面是MySQLdb/MySQL-Python默认写法
# engine = create_engine('mysql://root:mysql@127.0.0.1:3306/test', echo=True)

# 这里我使用的是PyMySQL
# echo=True是开启调试，这样当我们执行文件的时候会提示相应的文字
engine = create_engine('mysql+pymysql://root:mysql@127.0.0.1:3306/test', echo=True)
```

现在我们只是定义了表映射，而数据库里面是没有真实表的，这里我们使用Base类的metadata来帮我们自动创建表：

```python
Base.metadata.create_all(engine)
```

现在数据库里面已经有我们的两个表了。下面我们对这两个表进行常规操作

### 增删改查

对数据库的操作必须先创建一个session，增删改查操作都有这个session负责，首先我们先创建一个session工厂类，由它来负责后续的session创建

```python
from sqlalchemy.orm import sessionmaker

Session = sessionmaker(bind=engine)
```

#### 添加用户

```python
session = Session()  # 先使用工程类来创建一个session
ed_user = User(name='ed', fullname='Ed Jones', password='edspassword')
session.add(ed_user)
# 同时创建多个
session.add_all([
    User(name='wendy', fullname='Wendy Williams', password='foobar'),
    User(name='mary', fullname='Mary Contrary', password='xxg527'),
    User(name='fred', fullname='Fred Flinstone', password='blah')])
# 提交事务
session.commit()
```

#### 查询

在session上面调用`query()`方法会创建一个`Query`对象

```python
for user in session.query(User).order_by(User.id):
    print(user.name, user.fullname)

# 使用filter_by过滤
for name in session.query(User.name).filter_by(fullname='Ed Jones'):
    print(name)
# 使用sqlalchemy的SQL表达式语法过滤，可以使用python语句
for name in session.query(User.name).filter(User.fullname == 'Ed Jones'):
    print(name)

```

#### 删除

```python
session.delete(ed_user)
session.query(User).filter_by(name='ed').count()
```

### 一对多的关系映射

sqlalchemy使用`ForeignKey`来指明一对多的关系，比如一个用户可有多个邮件地址，而一个邮件地址只属于一个用户。那么就是典型的一对多或多对一关系。

在`Address`类中，我们定义外键，还有对应所属的user对象

```python
user_id = Column(Integer, ForeignKey('users.id'))
user = relationship("User", back_populates="addresses")
```

而在`User`类中，我们定义`addresses`属性

```python
addresses = relationship("Address", order_by=Address.id, back_populates="user")
```

注意两个类中都通过`relationship()`方法指明相互关系。

通过几个例子来操作一对多的关系映射

```python
# 先添加一个用户，并且给这个用户增加两个邮件地址
jack = User(name='jack', fullname='Jack Bean', password='gjffdd')
jack.addresses = [Address(email_address='jack@google.com'),
                  Address(email_address='j25@yahoo.com')]
session.add(jack)
session.commit()

# 查询
jack = session.query(User).filter_by(name='jack').one()
# 只有在调用jack.addresses时才会调用查询邮件地址的SQL，这个是典型的懒加载模式
jack.addresses

# join查询
session.query(User).join(Address).filter(Address.email_address == 'jack@google.com').all()
```

有时候我们不想使用懒加载，而是要强制一次性加载某个关联数据，那么可以使用`subqueryload`或者`joinedload`

```python
from sqlalchemy.orm import subqueryload

jack = session.query(User).options(subqueryload(User.addresses)).filter_by(name='jack').one()

# 推荐使用下面这种方案
from sqlalchemy.orm import joinedload

jack = session.query(User).options(joinedload(User.addresses)).filter_by(name='jack').one()
```

本篇文章只是对sqlalchemy的入门篇，更多细节请参考[官方教程](http://docs.sqlalchemy.org/en/rel_1_0/orm/tutorial.html)

