---
title: SQLAlchemy进阶
date: 2016-03-07 23:50:22 +0800
toc: true
categories: [ python ]
tags: [ python ]
---

前面一篇介绍了SQLAlchemy的入门，这里我讲讲它的进阶用法，其实主要是通过它来轻松实现一些复杂查询。

SQLAlchemy中的映射关系有四种,分别是一对多、多对一、一对一、多对多。接下来我将详细说明怎样去定义这四种关系，
然后再演示怎样通过这四种关系完成复杂的查询和更新。
<!-- more -->

## 一对多

表示一对多的关系时，在子表类中通过 foreign key (外键)引用父表类。

然后，在父表类中通过 relationship() 方法来引用子表的类：

```python
class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child")
    # 在父表类中通过 relationship() 方法来引用子表的类集合


class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))
    # 在子表类中通过 foreign key (外键)引用父表的参考字段
```

## 多对一

在一对多的关系中建立双向的关系，这样的话在对方看来这就是一个多对一的关系，
在子表类中附加一个`relationship()`方法，并且在双方的`relationship()`方法中使用`relationship.back_populates`方法参数：

```python
class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child", back_populates="parent")


class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))
    parent = relationship("Parent", back_populates="children")
    # 子表类中附加一个 relationship() 方法
    # 并且在(父)子表类的 relationship() 方法中使用 relationship.back_populates 参数
```

这样的话子表将会在多对一的关系中获得父表的属性

或者，可以在单一的`relationship()`方法中使用`backref`参数来代替`back_populates`参数，
推荐使用这种方式，可以少些几句话。

```python
class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child", backref="parent")


class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))
```

## 一对一

一对一就是多对一和一对多的一个特例,只需在relationship加上一个参数uselist=False替换多的一端就是一对一

从一对多转换到一对一:

```python
class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    child = relationship("Child", uselist=False, backref="parent")


class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('parent.id'))
```

从多对一转换到一对一:

```python
class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    child_id = Column(Integer, ForeignKey('child.id'))
    child = relationship("Child", backref=backref("parent", uselist=False))


class Child(Base):
    __tablename__ = 'child'
    id = Column(Integer, primary_key=True)
```

## 多对多

多对多关系需要一个中间关联表,通过参数`secondary`来指定。`backref`会自动的为子表类加载同样的`secondary`参数,
所以为了简洁起见仍然推荐这种写法：

```python
from sqlalchemy import Table, Text

post_keywords = Table('post_keywords', Base.metadata,
                      Column('post_id', Integer, ForeignKey('posts.id')),
                      Column('keyword_id', Integer, ForeignKey('keywords.id'))
                      )


class BlogPost(Base):
    __tablename__ = 'posts'
    id = Column(Integer, primary_key=True)
    body = Column(Text)
    keywords = relationship('Keyword', secondary=post_keywords, backref='posts')


class Keyword(Base):
    __tablename__ = 'keywords'
    id = Column(Integer, primary_key=True)
    keyword = Column(String(50), nullable=False, unique=True)
```

如果使用`back_populates`，那么两个都要定义：

```python
from sqlalchemy import Table, Text

post_keywords = Table('post_keywords', Base.metadata,
                      Column('post_id', Integer, ForeignKey('posts.id')),
                      Column('keyword_id', Integer, ForeignKey('keywords.id'))
                      )


class BlogPost(Base):
    __tablename__ = 'posts'
    id = Column(Integer, primary_key=True)
    body = Column(Text)
    keywords = relationship('Keyword', secondary=post_keywords, back_populates="parents")


class Keyword(Base):
    __tablename__ = 'keywords'
    id = Column(Integer, primary_key=True)
    keyword = Column(String(50), nullable=False, unique=True)
    parents = relationship('BlogPost', secondary=post_keywords, back_populates="keywords")
```

## 一些重要参数

`relationship()`函数接收的参数非常多，比如：`backref`，`secondary`，`primaryjoin`等等。
下面列举一下我用到的参数

* backref 在一对多或多对一之间建立双向关系,比如
* lazy:默认值是True, 懒加载
* remote_side: 表中的外键引用的是自身时, 如Node类,如果想表示多对一的树形关系, 那么就可以使用remote_side

```python
  class Node(Base):
    __tablename__ = 'node'
    id = Column(Integer, primary_key=True)
    parent_id = Column(Integer, ForeignKey('node.id'))
    data = Column(String(50))
    parent = relationship("Node", remote_side=[id])
```

* secondary: 多对多指定中间表关键字
* order_by: 在一对多的关系中,如下代码:

```python
class User(Base):
    # ...
    addresses = relationship(lambda: Address,
                             order_by=lambda: desc(Address.email),
                             primaryjoin=lambda: Address.user_id == User.id)
```

* cascade: 级联删除

```python
class Parent(Base):
    __tablename__ = 'parent'
    id = Column(Integer, primary_key=True)
    children = relationship("Child", cascade='all', backref='parent')


def delete_parent():
    session = Session()
    parent = session.query(Parent).get(2)
    session.delete(parent)
    session.commit()
```

不过不设置`cascade`，删除`parent`时，其关联的`chilren`不会删除，只会把`chilren`关联的`parent.id`置为空，
设置`cascade`后就可以级联删除`children`

## 对象的四种状态

对象在session中可能存在的四种状态包括：

* Transient：实例还不在session中，还没有保存到数据库中去，没有数据库身份，想刚创建出来的对象比如User()，仅仅只有mapper()与之关联
* Pending：用add()一个transient对象后，就变成了一个pending对象，这时候仍然没有flushed到数据库中去，直到flush发生。
* Persistent：实例出现在session中而且在数据库中也有记录了，通常是通过flush一个pending实例变成Persistent或者从数据库中querying一个已经存在的实例。
* Detached：一个对象它有记录在数据库中，但是不在任何session中，

## 关联查询

查询：<http://docs.sqlalchemy.org/en/rel_1_1/orm/query.html>

关联查询：<http://docs.sqlalchemy.org/en/rel_1_1/orm/query.html#sqlalchemy.orm.query.Query.join>

非常简单的关联查询，外键就一个，系统知道如何去关联：

```python
session.query(User).join(Address).filter(Address.email == "lzjun@qq.com").all()
```

指定ON字段：

```python
q = session.query(User).join(Address, User.id == Address.user_id)
```

多个join

```python
q = session.query(User).join("orders", "items", "keywords")
q = session.query(User).join(User.orders).join(Order.items).join(Item.keywords)
```

子查询JOIN：

```python
address_subq = session.query(Address).
filter(Address.email_address == 'ed@foo.com').
subquery()

q = session.query(User).join(address_subq, User.addresses)
```

join from:

```python
q = session.query(Address).select_from(User).
join(User.addresses).
filter(User.name == 'ed')
```

和下面的SQL等价：

```sql
SELECT address.*
FROM user
         JOIN address ON user.id = address.user_id
WHERE user.name = :name_1
```

左外连接，指定`isouter=True`，等价于` Query.outerjoin()`：

```python
q = session.query(Node).
join("children", "children", aliased=True, isouter=True).
filter(Node.name == 'grandchild 1')
```

