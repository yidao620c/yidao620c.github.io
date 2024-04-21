---
title: xpath入门笔记
date: 2015-05-19 14:01:07 +0800
toc: true
categories: [ 开发工具 ]
tags: [ 开发工具 ]
---

XPath即为XML路径语言（XML Path Language），它是一种用来确定XML文档中某部分位置的语言。
XPath基于XML的树状结构，提供在数据结构树中找寻节点的能力。
起初XPath的提出的初衷是将其作为一个通用的、介于XPointer与XSL间的语法模型。
但是XPath很快的被开发者采用来当作小型查询语言。

W3C网址： <http://www.w3schools.com/XPath/>
<!-- more -->

### 表示法

最常见的XPath表达式是路径表达式（XPath这一名称的另一来源）。

路径表达式是从一个XML节点（当前的上下文节点）到另一个节点、或一组节点的书面步骤顺序。
这些步骤以"／"字符分开，每一步有三个构成成分：

### 轴描述

节点测试（用于筛选节点位置和名称）

节点描述（用于筛选节点的属性和子节点特征）

一般情况下，我们使用简写后的语法。虽然完整的轴描述是一种更加贴近人类语言，
利用自然语言的单词和语法来书写的描述方式，但是相比之下也更加罗嗦。

### 三种表示法

1. 最简单的XPath如下：

```
/A/B/C
```

在这里选择所有符合规矩的C节点：C节点必须是B的子节点（B/C），
同时B节点必须是A的子节点（A/B），而A是这个XML文档的根节点（/A）。
此时的这种描述法类似于磁盘中文件的路径（URI），从盘符开始顺着一级一级的目录最终找到文件。

2. 这里还有一个复杂一些的例子，包含了全部构成成分（请详细的看）：

```
A//B/*[1]
```

此时选择的元素是：在B节点下的第一个节点（B/*[1]），不论节点的名称如何（*）；
而B节点必须出现在A节点内，不论和A节点之间相隔几层节点（//B）；
与此同时A节点还必须是当前节点的子节点（A，前边没有/）。

3. 最后一个常用的例子，在所有节点下查找：

```
//A/B/C/*[2]
```

### 轴语法

在未缩写语法里，两个上述范例可以写为：

```
/child::A/child::B/child::C
child::A/descendant-or-self::B/child::node()[1]
```

在XPath的每个步骤里，通过完整的轴描述（例如：child或descendant-or-self）进行明确的指定，
然后使用::，它的后面跟着节点测试的内容，例如上面范例所示的A以及node()。

### XPath轴

轴可定义相对于当前节点的节点集。
<table class="goodtable">
    <tr><td>ancestor</td><td>选取当前节点的所有先辈（父、祖父等）。</td></tr>
    <tr><td>ancestor-or-self</td><td>选取当前节点的所有先辈（父、祖父等）以及当前节点本身。</td></tr>
    <tr><td>attribute</td><td>选取当前节点的所有属性</td></tr>
    <tr><td>child</td><td>选取当前节点的所有子元素</td></tr>
    <tr><td>descendant</td><td>选取当前节点的所有后代元素（子、孙等）。</td></tr>
    <tr><td>descendant-or-self</td><td>选取当前节点的所有后代元素（子、孙等）以及当前节点本身。</td></tr>
    <tr><td>following</td><td>选取文档中当前节点的结束标签之后的所有节点。</td></tr>
    <tr><td>namespace</td><td>选取当前节点的所有命名空间节点。</td></tr>
    <tr><td>parent</td><td>选取当前节点的父节点。</td></tr>
    <tr><td>preceding</td><td>选取文档中当前节点的开始标签之前的所有节点。</td></tr>
    <tr><td>preceding-sibling</td><td>选取当前节点之前的所有同级节点。</td></tr>
    <tr><td>self</td><td>选取当前节点。</td></tr>
</table>

几个实例讲解：
<table class="goodtable">
    <tr><td>child::book</td><td>选取所有属于当前节点的子元素的 book 节点。</td></tr>
    <tr><td>attribute::lang</td><td>选取当前节点的 lang 属性。</td></tr>
    <tr><td>child::*</td><td>选取当前节点的所有子元素。</td></tr>
    <tr><td>attribute::*</td><td>选取当前节点的所有属性</td></tr>
    <tr><td>child::text()</td><td>选取当前节点的所有文本子节点</td></tr>
    <tr><td>child::node()</td><td>选取当前节点的所有子节点</td></tr>
    <tr><td>descendant::book</td><td>选取当前节点的所有book后代</td></tr>
    <tr><td>ancestor::book</td><td>选择当前节点的所有book先辈</td></tr>
    <tr><td>ancestor-or-self::book</td>选取当前节点的所有book先辈以及当前节点（如果此节点是 book 节点）<td></td></tr>
    <tr><td>child::*/child::price</td><td>选取当前节点的所有price孙节点。</td></tr>
</table>

### XPath 运算符

下面列出了可用在 XPath 表达式中的运算符：
<table class="goodtable">
    <tr><td>|</td><td>计算两个节点集</td><td>//book | //cd</td><td>返回所有拥有 book 和 cd 元素的节点集</td></tr>
    <tr><td>+</td><td>加法</td><td>6 + 4</td><td>10</td></tr>
    <tr><td>-</td><td>减法</td><td>6 – 4</td><td>2</td></tr>
    <tr><td>*</td><td>乘法</td><td>6 * 4</td><td>24</td></tr>
    <tr><td>div</td><td>除法</td><td>8 div 4</td><td>2</td></tr>
    <tr><td>=</td><td>等于</td><td>price=9.80</td><td>如果 price 是 9.80，则返回 true。如果 price 是 9.90，则返回 false。</td></tr>
    <tr><td>!=</td><td>不等于</td><td>price!=9.80</td><td>如果 price 是 9.90，则返回 true。如果 price 是 9.80，则返回 false。</td></tr>
    <tr><td>&lt;</td><td>小于</td><td>price&lt;9.80</td><td>如果 price 是 9.00，则返回 true。如果 price 是 9.90，则返回 false。</td></tr>
    <tr><td>&lt;=</td><td>小于或等于</td><td>price&lt;=9.80</td><td>如果 price 是 9.00，则返回 true。如果 price 是 9.90，则返回 false。</td></tr>
    <tr><td>&gt;</td><td>大于</td><td>price&gt;9.80</td><td>如果 price 是 9.90，则返回 true。如果 price 是 9.80，则返回 false。</td></tr>
    <tr><td>&gt;=</td><td>大于或等于</td><td>price&gt;=9.80</td><td>如果 price 是 9.90，则返回 true。如果 price 是 9.70，则返回 false。</td></tr>
    <tr><td>or</td><td>或</td><td>price=9.80 or price=9.70</td><td>如果 price 是 9.80，则返回 true。如果 price 是 9.50，则返回 false。</td></tr>
    <tr><td>and</td><td>与</td><td>price&gt;9.00 and price&lt;9.90</td><td>如果 price 是 9.80，则返回 true。如果 price 是 8.50，则返回 false。</td></tr>
</table>

### Xpath函数

有关数值的函数

 函数              | 说明                                                     
-----------------|--------------------------------------------------------
 fn:number(arg)  | 返回参数的数值。参数可以是布尔值、字符串或节点集。例子：number(‘100′)结果：100        
 fn:abs(num)     | 返回参数的绝对值。例子：abs(3.14)   结果：3.14例子：abs(-3.14)   结果：3.14 
 fn:ceiling(num) | 返回大于 num 参数的最小整数。例子：ceiling(3.14)  结果：4                
 fn:floor(num)   | 返回不大于 num 参数的最大整数。例子：floor(3.14)  结果：3                 
 fn:round(num)   | 把 num 参数舍入为最接近的整数。例子：round(3.14)  结果：3                 

有关字符串的函数

 函数                                   | 说明                                                                            
--------------------------------------|-------------------------------------------------------------------------------
 fn:string(arg)                       | 返回参数的字符串值。参数可以是数字、逻辑值或节点集。例子：string(314) 结果："314″                             
 fn:compare(comp1,comp2,collation)    | 如果 comp1 小于 comp2，则返回 -1。类推例子：compare(‘ghi’, ‘ghi’) 结果：0                      
 fn:concat(string,string,…)           | 返回字符串的拼接。例子：concat(‘XPath ‘,’is ‘,’FUN!’) 结果：’XPath is FUN!’                  
 fn:substring(string,start,len)       | 返回从start位置开始的指定长度的子字符串。第一个字符的下标是 1。例子：substring(‘Beatles’,1,4) 结果：’Beat’      
 fn:string-length(string)             | 返回指定字符串的长度。如果没有 string 参数，则返回当前节点的字符串值的长度。例子：string-length(‘Beatles’) 结果：7    
 fn:normalize-space(string)           | 删除开头和结尾空白，并把内部所有空白序列替换为一个，然后返回结果。例子：normalize-space(‘ The XML ‘) 结果：’The XML’ 
 fn:upper-case(string)                | 把 string 参数转换为大写。例子：upper-case(‘The XML’) 结果：’THE XML’                        
 fn:lower-case(string)                | 把 string 参数转换为小写。例子：lower-case(‘The XML’) 结果：’the xml’                        
 fn:contains(string1,string2)         | 如果 string1 包含 string2，则返回 true，否则返回 false。例子：contains(‘XML’,’XM’) 结果：true     
 fn:starts-with(string1,string2)      | 如果 string1 以 string2 开始，则返回 true，否则返回 false。例子：starts-with(‘XML’,’X’) 结果：true 
 fn:ends-with(string1,string2)        | 如果 string1 以 string2 结尾，则返回 true，否则返回 false。例子：ends-with(‘XML’,’X’) 结果：false  
 fn:substring-before(string1,string2) | 返回 string2 在 string1 中出现之前的子字符串。例子：substring-before(’12/10′,’/’) 结果：’12’      
 fn:substring-after(string1,string2)  | 返回 string2 在 string1 中出现之后的子字符串。例子：substring-after(’12/10′,’/’) 结果：’10’       
 fn:matches(string,pattern)           | 如果 string 参数匹配指定的模式，则返回 true，否则返回 false。例子：matches("Merano", "ran") 结果：true   

更多函数请参考： <http://www.w3school.com.cn/xpath/xpath_functions.asp>

### 我自己实际工作中使用过的XPath实例：

```
* //span/../.././span
* //bookstore/book[last()]
* /DocText/WithQuads/Page/Word
* record[field[@id='220' and @value='Red'] and field[@id='221' and @value='Large']]
* /Root//Person[contains(Blog,'cn') and contains(@ID,'01')]
* //tr[td[1] and td[2][contains(text(), "512M")]]
* //td/following-sibling::td[1]
* //td/preceding-sibling::td[1]
* //td[starts-with(text(), "%s") and contains(text(), "disk:%sMB")]/following-sibling::td[2][contains(text(), "%s")]
* //a/../following-sibling::td[8]/a[2]
* //a[@class="pushstate-link" and not(@rel)]
* //a[@rel="next" and not(contains(@class, "disabled"))]
```

看完前面部分，这些的含义应该很容易可以看懂了。恭喜你，基本的XPath已经没问题了！

### chrome插件PsychoXPath

最后我还推荐一个chrome浏览器中很好用的xpath插件，名字叫PsychoXPath。

插件地址：[PsychoXPath](https://chrome.google.com/webstore/detail/psychoxpath/bpnigkcdmnofjkmojlopmelmhgpbndog)

基本使用方法，以google的首页"Google 搜索"按钮为例：

高亮模式：

1. 先按F12打开chrome浏览器的调试窗口，然后通过邮件审查元素找到"Google 搜索"按钮，查看对应的html代码。

![](https://xnstatic-1253397658.file.myqcloud.com/x002.png)

*. 然后右键选择PsychoXPath->Test XPath(Highlight)

![](https://xnstatic-1253397658.file.myqcloud.com/x006.png)

*. 之后输入XPath路径

![](https://xnstatic-1253397658.file.myqcloud.com/x004.png)

*. 结果如下，被找到的页面元素会被高亮显示：

![](https://xnstatic-1253397658.file.myqcloud.com/x005.png)

*. 控制台模式：

还可以在控制台中调试xpath，这个跟上面同样道理。只是这次选择的是PsychoXPath->Test XPath(Console)模式就行了。

具体我就不再细说了，使用还是很容易的。
