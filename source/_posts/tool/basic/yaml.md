---
title: yaml入门笔记
date: 2015-05-22 13:53:45 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

YAML是一个可读性高，用来表达资料序列的格式。YAML参考了其他多种语言，包括：XML、C语言、Python、Perl以及电子邮件格式RFC2822。
目前已经有数种编程语言或脚本语言支援（或者说解析）这种语言。

最新版本为1.2，官方说明地址： <http://www.yaml.org/spec/1.2/spec.html>

使用方式：作为配置文件，数据交换格式，序列化对象存储，测试数据文件，
<!-- more -->

一个简单的示例：

`#` 表示注释，从这个字符一直到行尾，都会被解析器忽略。

```yaml

---
# 这一行是注释部分，不会被解析
receipt: Oz-Ware Purchase Invoice
date: 2007-08-06
customer:
  given: Dorothy
  family: Gale

items:
  - part_no: A4786
    descrip: Water Bucket (Filled)
    price: 1.47
    quantity: 4

  - part_no: E1628
    descrip: High Heeled "Ruby" Slippers
    price: 100.27
    quantity: 1

bill-to: &id001
  street: |
    123 Tornado Alley
    Suite 16
  city: East Westville
  state: KS

ship-to: *id001

specialDelivery: >
  Follow the Yellow Brick
  Road to the Emerald City.
  Pay no attention to the
  man behind the curtain.
```

### 基本技巧

#### 缩进规则

YAML使用一个固定的缩进风格表示数据层结构关系。缩进只能用空格（不能用tab），缩排中空格数目不重要，
只要相同阶层的元素左侧对齐就可以了，建议缩进用2个或4个空格。

#### 列表

使用 - 表示，也就是用短杠+空白字符作为起始。

另外还有一种内置格式（inline format）可以选择──用方括号围住，并用逗号+空白区隔（类似JSON的语法）。
比如：shopping: [milk, pumpkin pie, eggs, juice]

#### 对象

```
# 区块形式
person:
  name: John Smith
  age: 33

# 内置形式
person: {name: John Smith, age: 33}
```

#### 重复元素

使用&id001先标记，然后后面用*id001指针引用

```yaml
# & 的作用，它表示一个"锚点标记"，其它节点可以使用"*"或"<<: *"来引用它的值
node3: &node3
  a: 001
  b: 002

# * 的作用，指node4的内容与node3完全一致
node4:
  *node3

# <<: * 的作用，指node5的内容包含但不完全相同于node3的值。
node5:
  <<: *node3
  c: 003
```

一个更加详细的例子：

```yaml

---
- step: &id001                    # 定义锚点标记&id001
    instrument: Lasik 2000
    pulseEnergy: 5.4
    pulseDuration: 12
    repetition: 1000
    spotSize: 1mm

- step:
    <<: *id001                  # 合并键值：使用在锚点标签定义的內容
    spotSize: 2mm         # 覆盖"spotSize"键值

- step:
    <<: *id001                  # 合并键值：使用在锚点标签定义的內容
    pulseEnergy: 500.0       # 覆盖键值
    alert: >                    # 加入其他键值
      warn patient of
      audible pop
```

#### 需要换行书写的字符串，两种方式

再次强调，字符串不需要包在引号之内。

保存新行(Newlines preserved)

```
poetry: |
  There once was a man from Darjeeling
  Who got on a bus bound for Ealing
      It said on the door
      "Please don't spit on the floor"
  So he carefully spat on the ceiling
```

根据设定，前方的引领空白符号（leading white space）必须对齐，以便和其他资料或是行为（如范例中的缩排）明显区分。

折叠新行(Newlines folded)

```
poetry: >
  Wrapped text
  will be folded
  into a single
  paragraph
  
  Blank lines denote
  paragraph breaks
```

和保存新行不同的是，换行字元会被转换成空白字符，空行被转换成换行，而前导空白字符则会被自动消去，上面会变成两行。

#### 混合使用

在列表中使用映射

```
- {name: John Smith, age: 33}
- name: Mary Smith
  age: 27
```

在映射中使用列表

```
men: [John Smith, Bill Jones]
women:
  - Mary Smith
  - Susan Williams
```

#### 数据类型

布尔值用true和false表示：

```
isSet: true
```

null用~表示：

```
parent: ~
```

强制转换数据类型：

```
ee: !!str 123
ff: !!str true
```

转为 Json 如下：`{ e: '123', f: 'true' }`

#### 多个文件

当一个yaml文件内包含多个yaml数据文档时(multi documents in a stream)，
可以用`---`表示每个文档的开始。同时可以用`...`作为文档结束，`...`可以省略。

典型的应用是Log文件每天的记录：

```yaml

---
time: 20:03:20
player: Sammy Sosa
action: strike (miss)
...

---
time: 20:03:47
player: Sammy Sosa
action: grand slam
```

### 更多资源

<http://www.yaml.org/>
