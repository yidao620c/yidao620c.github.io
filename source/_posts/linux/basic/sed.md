---
title: sed命令笔记
date: 2016-09-20 22:22:22 +0800
toc: true
categories: [ linux ]
tags: [ linux ]
---

在linux中通常要进行文本处理，sed是一个非常强大的文本处理命令工具。
配合正则表达式可以进行文本搜索、替换、插入、删除等操作。
sed基本上就是正则模式匹配，所以你的正则表达式要比较强才能玩的好它。

sed是一种新型的，非交互式的编辑器。它能执行与编辑器vi和ex相同的编辑任务。
sed编辑器没有提供交互使用方式，使用者只能在命令行输入编辑命令、指定文件名，然后在屏幕上查看输出。
sed编辑器没有破坏性，它不会修改文件，除非指定-i选项，默认情况下，所有的输出行都被打印到屏幕上。

这里只介绍最常用的一些用法，如果要看sed全部东西，
请参考[sed参考手册](http://www.gnu.org/software/sed/manual/sed.html)
<!-- more -->

## 正则式

介绍一下正则表达式的一些最基本的东西：

- `^`   表示一行的开头。如：/^#/ 以#开头的匹配。
- `$`   表示一行的结尾。如：/}$/ 以}结尾的匹配。
- `\<`  表示词首。 如 \<abc 表示以 abc 为首的詞。
- `\>`  表示词尾。 如 abc\> 表示以 abc 結尾的詞。
- `.`   表示任何单个字符。
- `*`   表示某个字符出现了0次或多次。
- `[ ]` 字符集合。 如：[abc]表示匹配a或b或c，还有[a-zA-Z]表示匹配所有的26个字符。如果其中有^表示反，如[^a]表示非a的字符

## s命令替换

```bash
sed "s/my/your/g" test.txt     # 将my全部修改成your
sed -i "s/my/your/g" test.txt  # -i选项直接修改文件
sed 's/^/#/g' test.txt         # 每一行前面加注释#
sed 's/<[^>]*>//g' test.html   # 删除所有的html标签
sed "3,6s/my/your/g" test.txt  # 只替换第3到第6行的文本
sed 's/old/new/1' test.txt     # 只替换每行的第1个
sed 's/s/S/3g' test.txt        # 只替换每行第3个及其以后的
```

## 执行多次匹配

```bash
# 第1到第3行的my换成your，从第3行开始This换成That
sed -e '1,3s/my/your/g' -e '3,$s/This/That/g' my.txt
```

我们可以使用&来当做被匹配的变量，然后可以在被匹配项的两边加中括号：

```bash
sed 's/my/[&]/g' my.txt
```

## 分组匹配

和python类似，使用括号来进行分组，后面可以通过\1,\2来引用分组：

```bash
sed 's/This is my \([^,]*\),.*is \(.*\)/\1:\2/g' my.txt
```

## a和i命令添加行

a命令就是append后面一行添加，i命令就是insert前面一行插入，它们是用来添加行的。如

```bash
# 第1行前插入
sed "1 i aaaaaaaaaaaaaaaa" my.txt
sed "/fish/a bbbbbbbbbbbbbbbbbbbbb" my.txt
# 最后一行添加
sed "$ a aaaaaaaaaaaaaaaa" my.atxt
# 某行后面插入多行字符串
sed -i'/bind_address/a\skip-name-resolve\nlower_case_table_names=1' /etc/mysql/my.cnf
```

## c命令替换行

c 命令是替换匹配行：

```bash
sed "/fish/c qqqqqqqqqqqqqqqqqqqqqqqq" my.txt
```

## d命令删除行

```bash
sed -i '/fish/d' my.txt
```

## p命令打印

你可以把这个命令当成grep式的命令：

```bash
sed -n '/fish/p' my.txt
# 从一个匹配行打印到另一个匹配行
sed -n '/dog/,/fish/p' my.txt
# 从第一行打印到匹配fish成功的那一行
sed -n '1,/fish/p' my.txt
```

## sed更多范例

```bash
sed '/north/p' datafile                      # 命令p是打印命令,默认情况下是打印所有输入行；选项-n是用于取消默认的打印操作。
sed  -n '/north/p' datafile                  # 打印datafile中含有north模式的行，只打印匹配到的行。
sed  '3,$d' datafile                         # 删除从第3行到最后一行的内容。
sed  's/west/north/g' datafile               # 所有行中的west替换成north.若无g,则每行的第一个west被替换。
sed  's/[0-9][0-9]$/&.5/' datafile           # & 它代表在查找串中匹配到的内容，这个例子中，所有两位数结尾的行后面都被加上.5
sed  -n 's/\(Mar\)got/\1ianne/p' datafile    # 包含在圆括号里的模式Mar作为标签1 保存于特定的寄存器中。替换串可通过\1引用它。则Margot被替换成Marianne。
sed 's#3#88#g' datafile                      # 可以设置新的分隔符为#
sed -n '/west/,/east/p' datafile             # 打印west行 到 east行之间的所有行
sed -e '1,3d' -e 's/Mar/Apr/' datafile       # -e多重编辑,先删除1~3行，然后进行替换。
sed '/Suan/r newfile ' datafile              # r命令是读取指定文件。如果再文件datafile 的某一行匹配到Suan，就在该行后面读入文件newfile的内容。
sed -n '/north/w newfile' datafile           # w命令是写命令，将匹配的行写入newfile文件中。
sed '/north/a\ooooooooooooo' datafile        # 匹配north的下一行追加一串ooooo字符。
sed '/north/i\ooooooooooooo' datafile        # 匹配north的前一行插入一串ooooo字符。
sed '/north/c\ooooooooooooo' datafile        # 匹配north行，将ooooo替换该行。
sed '/north/{n; s/AM/PM/;}' datafile         # n 命令表示下一条命令。sed使用该命令获取输入文件的下一行，并将其读入到模式缓冲区中，任何sed命令都将应用到匹配行的下一行上。此处命令的含义是：匹配到含有north后，将输入行下移一行 然后将下一行文本中的AM替换成PM。可以认为n命令是跳跃一行。
sed '1,3y/abc/ABC/' datafile                 # y 命令是一对一转换，此处是将1~3行中a->A b->B c->C转换。
sed '5q' datafile                            # 打印完第五行只后，q命令让sed程序退出.
sed '/Lew/{ s/Lew/John/ ; q;} ' datafile     # 在某行匹配到模式 Lewis时，s表示先用John替换Lewis，然后q命令让sed程序退出。
sed -e '/north/h' -e '$G'  datafile          # 在匹配到north行时,将该行复制到暂存缓冲区中，然后再匹配最后一行后取出暂存缓冲区的内容追加在文件尾部。
sed -n '/a/ {n;p}' test.log                  # 打印出符合开头是a的记录的下一行
```
