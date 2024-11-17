---
title: 语法解析器ANTLR4简易教程
date: 2024-06-23 11:23:35 +0800
toc: true
categories: [ 开发工具 ]
tags: [ Antlr4 ]
---

Antlr (ANother Tool for Language Recognition) 是一个强大的跨语言语法解析器，可以用来读取、处理、执行或翻译结构化文本或二进制文件。
它被广泛用来构建语言，工具和框架。。ANTLR 根据语法定义生成解析器，解析器可以构建和遍历解析树。

所有编程语言的语法，都可以用ANTLR来定义。ANTLR提供了大量的官方 [grammar](https://github.com/antlr/grammars-v4/) 示例，
包含了各种常见语言，比如Java、SQL、Javascript、PHP等等。

**谁在使用**

* Twitter搜索使用ANTLR进行语法分析，每天处理超过20亿次查询；
* Hadoop生态系统中的Hive、Pig、数据仓库和分析系统所使用的语言都用到了ANTLR；
* Lex Machina将ANTLR用于分析法律文本；Oracle公司在SQL开发者IDE和迁移工具中使用了ANTLR；
* NetBeans公司的IDE使用ANTLR来解析C++；
* Hibernate对象-关系映射框架（ORM）使用ANTLR来处理HQL语言
* 其他还有Oracle、Presto、Elasticsearch、Spark

官网地址：https://www.antlr.org

<!-- more -->

## 基本概念

![img.png](https://xnstatic-1253397658.file.myqcloud.com/antlr4-01.png)

1. 词法分析器 (Lexer)词法分析是指在计算机科学中，将字符序列转换为单词(Token)的过程。词法分析器(Lexer)一般是用来供语法解析器(Parser)调用的。
2. 语法解析器 (Parser) 语法解析器通常作为编译器或解释器出现。它的作用是进行语法检查，并构建由输入单词(Token)组成的数据结构(即抽象语法树)。
语法解析器通常使用词法分析器(Lexer)从输入字符流中分离出一个个的单词(Token)，并将单词(Token)流作为其输入。实际开发中，语法解析器可以手工编写，也可以使用工具自动生成。
3. 抽象语法树 (Abstract Syntax Tree,AST) 抽象语法树是源代码结构的一种抽象表示，它以树的形状表示语言的语法结构。抽象语法树一般可以用来进行代码语法的检查，
代码风格的检查，代码的格式化，代码的高亮，代码的错误提示以及代码的自动补全等等。

## 其他常见的语法分析器
**JavaCC**

JavaCC，即Java Cmopiler Compiler，为了简化基于Java语言的词法分析器或者语法分析器的开发，Sun公司的开发人员开发了JavaCC(Java Compiler Compiler)。
JavaCC是一个基于Java语言的分析器的自动生成器。用户只要按照JavaCC的语法规范编写JavaCC的源文件，然后使用JavaCC的编译器编译，
就能够生成基于Java语言的某种特定语言的分析器。JavaCC已经成为最受欢迎的Java解析器创建工具。

**YACC**

YACC(Yet Another Compiler-Compiler): 1975 年由贝尔实验室 Mike Lesk & Eric Schmidt 开发，UNIX 标准实用工具 (utility)。

**SqlParser** 

位于Alibaba的Druid库中，只能解析sql语句，功能比较单一。

## 安装方式
有多种安装Antlr4的方式：

1. 使用pip方式安装antlr4-tools
2. Maven工程中引入antlr4的依赖
3. IDEA中安装antlr4的插件

我平时都是喜欢用maven开发代码，因此选择第二种maven方式。
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.xncoding</groupId>
    <artifactId>antlr4-demo</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <name>antlr4-demo</name>
    <description>antlr4 java source</description>
    <url>https://maven.apache.org</url>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.antlr</groupId>
            <artifactId>antlr4</artifactId>
            <version>4.13.2</version>
        </dependency>
    </dependencies>

    <build>
        <finalName>antlr4</finalName>
        <sourceDirectory>src/main/java</sourceDirectory>
        <plugins>
            <plugin>
                <groupId>org.antlr</groupId>
                <artifactId>antlr4-maven-plugin</artifactId>
                <version>4.13.2</version>
                <executions>
                    <execution>
                        <id>antlr4</id>
                        <goals>
                            <goal>antlr4</goal>
                        </goals>
                        <configuration>
                            <includes>
                                <include>**/*.g4</include>
                            </includes>
                            <listener>true</listener>
                            <visitor>true</visitor>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

## 基本使用步骤
通过一个简单额hello world的例子来介绍Antlr4语法解析器开发的基本使用步骤。

### 定义一个G4文件
需要创建一个.g4文件，用于定义词法分析器（lexer）和语法解析器(Parser)。具体语法参见官方文档。下面是一个简单的例子：Hello.g4。
```antlrv4
// file Hello.g4
// Define a grammar called Hello
grammar Hello; // 1. grammer name
@header { package pers.me.expression.parser; } // 2. java package
r  : 'hello' ID ;         // 3. match keyword hello followed by an identifier
ID : [a-z]+ ;             // match lower-case identifiers
WS : [ \t\r\n]+ -> skip ; // 4. skip spaces, tabs, newlines
```

1. 定义了 grammar 的名字，名字需要与文件名对应
2. 定义生成的Java类的package
3. r 定义的语法，会使用到下方定义的正则表达式词法
4. 定义了空白字符，后面的 skip 是一个特殊的标记，标记空白字符会被忽略

### 生成代码
执行 `mvn clean compile` 可自动生成Antlr的代码。

### 测试
我们可以利用下面这段代码来测试一下ParseTree。
```java
public class HelloTest {
    public static void main(String[] args) throws Exception {
        HelloLexer lexer = new HelloLexer(CharStreams.fromString("hello world"));
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        HelloParser parser = new HelloParser(tokens);
        ParseTree tree = parser.r();
        System.out.println(tree.toStringTree(parser));
    }
}
```
运行上面的代码可以得到如下输出，程序识别出输入的字符串符合r的语法。
```
(r hello world)
```

## Listener和Visitor
ANTLR提供了两种方法来访问ParseTree：

1. 一种是通过Parse-Tree Listener的方法
2. 另一种是通过Parse-Tree Visitor的方法

Listener有以下特点：

1. 访问AST的所有节点
2. 重写（Override）进入时（enterXXX方法）和退出时（exitXXX方法）要执行的方法
3. 要重写的方法没有返回值，因此需要在属性中保留所需的值

Visitor有以下特点：

1. 并非所有 AST 节点都被访问
2. 根据目的重写进入节点时要执行的过程（visitXXX方法）
3. 重写方法有一个返回值，因此您不必在属性中保存所需的值

最大的区别是Listener会自动访问 AST 的所有节点，而Visitor如果要访问当前节点的子树，则需要手工实现。
Visitor 较为简单方便，继承 HelloBaseVisitor 类即可，内部的方法与 g4 文件定义相对应，对照看即可理解。实现了 visitor 之后，
就可以完成一个简单的自定义解析器了。自动生成的HelloBaseVisitor.java如下：
```java
import org.antlr.v4.runtime.tree.AbstractParseTreeVisitor;
/**
 * This class provides an empty implementation of {@link HelloVisitor},
 * which can be extended to create a visitor which only needs to handle a subset
 * of the available methods.
 *
 * @param <T> The return type of the visit operation. Use {@link Void} for
 * operations with no return type.
 */
public class HelloBaseVisitor<T> extends AbstractParseTreeVisitor<T> implements HelloVisitor<T> {
   /**
    * {@inheritDoc}
    *
    * <p>The default implementation returns the result of calling
    * {@link #visitChildren} on {@code ctx}.</p>
    */
   @Override public T visitR(HelloParser.RContext ctx) { return visitChildren(ctx); }
}
```

## 计算器的实现
接下来进入实战环节，我们来实现一个简单的计算器。

### 定义语法和词法
创建Expression.g4。
```antlrv4
// 定义了 grammar 的名字，名字需要与文件名对应
grammar Expression;
@header {
// 定义package
package pers.me.expression.parser;
}

/**
 * parser
 * calc 和 expr 就是定义的语法，会使用到下方定义的词法
 * 注意 # 后面的名字，是可以在后续访问和处理的时候使用的。
 * 一个语法有多种规则的时候可以使用 | 来进行配置。
 */

 calc
     : (expr)* EOF                                    # calculationBlock
     ;
// 四则运算分为了两个非常相似的语句，这样做的原因是为了实现优先级，乘除是优先级高于加减的。
 expr
    : BR_OPEN expr BR_CLOSE                           # expressionWithBr
    | sign=(PLUS|MINUS)? num=(NUMBER|PERCENT_NUMBER)  # expressionNumeric
    | expr op=(TIMES | DIVIDE) expr                   # expressionTimesOrDivide
    | expr op=(PLUS | MINUS) expr                     # expressionPlusOrMinus
   ;

/**
 * lexer
 */

BR_OPEN:   '(';
BR_CLOSE:  ')';
PLUS:      '+';
MINUS:     '-';
TIMES:     '*';
DIVIDE:    '/';
PERCENT:   '%';
POINT:     '.';

// 定义百分数
PERCENT_NUMBER
    : NUMBER ((' ')? PERCENT)
    ;

NUMBER
    : DIGIT+ ( POINT DIGIT+ )?
    ;

DIGIT
   : [0-9]+
   ;
   
// 定义了空白字符，后面的 skip 是一个特殊的标记，标记空白字符会被忽略
SPACE
   : ' ' -> skip
   ;
```

### 实现Visitor
生成Java文件之后，我们来实现自己的Visitor，用于支持BigDecimal。
```java
public class BigDecimalCalculationVisitor extends ExpressionBaseVisitor<BigDecimal> {

    /**
     * 100
     */
    private static final BigDecimal HUNDRED = BigDecimal.valueOf(100);

    /**
     * DECIMAL128のMathContext (桁数34、RoundingMode.HALF_EVEN)
     */
    private static final MathContext MATH_CONTEXT = MathContext.DECIMAL128;

    @Override
    public BigDecimal visitCalculationBlock(ExpressionParser.CalculationBlockContext ctx) {
        BigDecimal calcResult = null;
        for (ExpressionParser.ExprContext expr : ctx.expr()) {
            calcResult = visit(expr);
        }
        return calcResult;
    }

    @Override
    public BigDecimal visitExpressionTimesOrDivide(ExpressionParser.ExpressionTimesOrDivideContext ctx) {
        BigDecimal left = visit(ctx.expr(0));
        BigDecimal right = visit(ctx.expr(1));
        switch (ctx.op.getType()) {
            case ExpressionLexer.TIMES: return left.multiply(right, MATH_CONTEXT);
            case ExpressionLexer.DIVIDE: return left.divide(right, MATH_CONTEXT);
            default: throw new RuntimeException("unsupported operator type");
        }
    }

    @Override
    public BigDecimal visitExpressionPlusOrMinus(ExpressionParser.ExpressionPlusOrMinusContext ctx) {
    // 此处加减法的实现类似上面的乘除法，省略
    }

    @Override
    public BigDecimal visitExpressionWithBr(ExpressionParser.ExpressionWithBrContext ctx) {
        return visit(ctx.expr());
    }

    @Override
    public BigDecimal visitExpressionNumeric(ExpressionParser.ExpressionNumericContext ctx) {
        BigDecimal numeric = numberOrPercent(ctx.num);
        if (Objects.nonNull(ctx.sign) && ctx.sign.getType() == ExpressionLexer.MINUS) {
            return numeric.negate();
        }
        return numeric;
    }

    private BigDecimal numberOrPercent(Token num) {
        String numberStr = num.getText();
        switch (num.getType()) {
            case ExpressionLexer.NUMBER: return decimal(numberStr);
            case ExpressionLexer.PERCENT_NUMBER: return decimal(numberStr.substring(0, numberStr.length() - 1).trim()).divide(HUNDRED, MATH_CONTEXT);
            default: throw new RuntimeException("unsupported number type");
        }
    }

    private BigDecimal decimal(String decimalStr) {
        return new BigDecimal(decimalStr);
    }

}
```

### 调用解析器
在Calculator类中调用Expression。
```java
public class Calculator {
    public BigDecimal execute(String expression) {
        CharStream cs = CharStreams.fromString(expression);
        ExpressionLexer lexer = new ExpressionLexer(cs);
        CommonTokenStream tokens = new CommonTokenStream(lexer);
        ExpressionParser parser = new ExpressionParser(tokens);
        ExpressionParser.CalcContext context = parser.calc();
        BigDecimalCalculationVisitor visitor = new BigDecimalCalculationVisitor();
        return visitor.visit(context);
    }
}
```

### 测试
最后，我们用一个jUnit来测试一下我们的计算器。
```java
public class CalculatorUnitTest {

    private final Calculator calculator = new Calculator();

    @DisplayName("Test Calculator")
    @ParameterizedTest
    @CsvSource({
        "1 + 2, 3",
        "3 - 2, 1",
        "2 * 3, 6",
        "6 / 3, 2",
        "6 / (1 + 2) , 2",
        "50%, 0.5",
        "100 * 30%, 30.0"
    })
    void testCalculation(String expression, String expected) {
        assertEquals(expected, calculator.execute(expression).toPlainString());
    }

}
```

##  推荐阅读
1. [Antlr4官方指南](https://github.com/antlr/antlr4/blob/master/doc/index.md)
2. [Antlr4官方示例:Grammars-v4](https://github.com/antlr/grammars-v4)
