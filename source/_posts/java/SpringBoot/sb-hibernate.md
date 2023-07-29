---
title: SpringBoot系列 - 集成Hibernate
date: 2017-07-03 14:41:39 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

Hibernate与MyBatis都是流行的持久层开发框架，前一遍介绍了怎样在SpringBoot中集成MyBatis，本篇来介绍如何集成Hibernate作为DAO层。

Hibernate 是一个高性能的对象/关系映射（ORM）持久化存储和查询的服务，不仅负责从Java类到数据库表的映射
（还包括从Java数据类型到SQL数据类型的映射），还提供了面向对象的数据查询检索机制，从而极大地缩短了手动处理SQL和JDBC上的开发时间。
同时，Hibernate还实现了JPA规范，在SpringBoot中，JPA的默认实现就是使用的Hibernate。
<!-- more -->

## JPA和Hibernate

在讲解如何在SpringBoot中使用Hibernate框架之前，先要弄清几个基本概念以及它们之间的关系。

1. JPA
1. Hibernate
1. Spring Data
1. Spring Data JPA

### JPA

JPA(Java Persistence API)是Sun官方提出的Java持久化规范，它为Java开发人员提供了一种对象/关系映射工具来管理Java应用中的关系数据。

上面已经描述很清楚了，JPA是一种规范和标准，也就是类似于Java中的接口，由Sun公司官方定义，但是并没有指定谁来实现。

现在几个主要的JPA实现技术有Hibernate、EclipseLink、OpenJPA等。

### Hibernate

Hibernate是一个开放源代码的对象关系映射框架，它对JDBC进行了非常轻量级的对象封装，它将POJO与数据库表建立映射关系，是一个全自动的ORM框架。
Hibernate可以自动生成SQL语句、自动执行，从而使得Java程序员可以随心所欲的使用对象编程思维来操纵数据库。

实际上，JPA规范制定过程中就是借鉴了Hibernate等这些开源的持久框架，也就是说Hibernate的出现比JPA还要早些。

在Hibernate中使用的注解就是JPA注解，Hibernate实现了JPA规范。

### Spring Data

Spring Data是一个用于简化数据库访问，并支持云服务的开源框架。其主要目标是使得数据库的访问变得方便快捷，
并支持map-reduce框架和云计算数据服务。此外，它还支持基于关系型数据库的数据服务，如Oracle RAC等。

所以Spring Data本身就是一个开源的框架。

### Spring Data JPA

上面说过Spring Data是一个开源框架，在这个框架中Spring Data JPA只是这个框架中的一个模块。

它可以极大的简化JPA的写法，可以在几乎不用写实现的情况下，实现对数据的访问和操作。除了CRUD外，还包括如分页、排序等一些常用的功能。

好了，现在你应该对这些概念有一个基本认识了，接下来进入正题，演示如何在SpringBoot中使用Hibernate。

## maven依赖

第一步添加maven依赖：

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
<groupId>mysql</groupId>
<artifactId>mysql-connector-java</artifactId>
<version>${mysql-connector.version}</version>
<scope>runtime</scope>
</dependency>
<dependency>
<groupId>com.alibaba</groupId>
<artifactId>druid</artifactId>
<version>${druid.version}</version>
</dependency>
```

注意，使用mysql数据库来演示的话还要添加数据库驱动包，另外还添加了druid连接池。

## 准备数据库

如何安装mysql，配置用户名和密码之类我就不在这里讲了，请自行google。

下面是我的schema.sql文件，创建一个pos数据库，并创建一个article表，插入几条初始数据：

```sql
-- Dumping database structure for concretepage
CREATE
DATABASE IF NOT EXISTS `pos` default charset utf8 COLLATE utf8_general_ci;
SET
FOREIGN_KEY_CHECKS=0;
USE
`pos`;
-- Dumping structure for table concretepage.articles
CREATE TABLE IF NOT EXISTS `articles`
(
    `article_id` int
(
    5
) NOT NULL AUTO_INCREMENT,
    `title` varchar
(
    200
) NOT NULL,
    `category` varchar
(
    100
) NOT NULL,
    PRIMARY KEY
(
    `article_id`
)
    ) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8 COMMENT='文章表';

-- Dumping data for table concretepage.articles: ~3 rows (approximately)
INSERT INTO `articles` (`article_id`, `title`, `category`)
VALUES (1, 'Java Concurrency', 'Java'),
       (2, 'Hibernate HQL ', 'Hibernate'),
       (3, 'Spring MVC with Hibernate', 'Spring');

```

注意，Hibernate可以根据实体类自动创建表，本篇我不介绍这个功能，还是手动创建表结构。

## 修改配置文件

数据库创建好后，修改application.yml配置文件，新增如下数据库连接配置：

```yaml
###################  spring配置  ###################
spring:
  profiles:
    active: dev
  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
        new_generator_mappings: false
        format_sql: true
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/pos?serverTimezone=UTC&useSSL=false&autoReconnect=true&tinyInt1isBit=false&useUnicode=true&characterEncoding=utf8
    username: root
    password: 123456
```

## 实体类

article表对应的实体类如下：

```java
@Entity
@Table(name = "articles")
public class Article implements Serializable {
    private static final long serialVersionUID = 1L;
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "article_id")
    private int articleId;
    @Column(name = "title")
    private String title;
    @Column(name = "category")
    private String category;

    // 省略getter/setter
}
```

## DAO设计

先创建一个DAO的接口类：

```java
public interface IArticleDAO {
    List<Article> getAllArticles();

    Article getArticleById(int articleId);

    void addArticle(Article article);

    void updateArticle(Article article);

    void deleteArticle(int articleId);

    boolean articleExists(String title, String category);
}
```

然后写一个实现类，在这类中注入`EntityManager`

```java
@Repository
public class ArticleDAO implements IArticleDAO {
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public Article getArticleById(int articleId) {
        return entityManager.find(Article.class, articleId);
    }

    @SuppressWarnings("unchecked")
    @Override
    public List<Article> getAllArticles() {
        String hql = "FROM Article as atcl ORDER BY atcl.articleId";
        return (List<Article>) entityManager.createQuery(hql).getResultList();
    }

    @Override
    public void addArticle(Article article) {
        entityManager.persist(article);
    }

    @Override
    public void updateArticle(Article article) {
        Article artcl = getArticleById(article.getArticleId());
        artcl.setTitle(article.getTitle());
        artcl.setCategory(article.getCategory());
        entityManager.flush();
    }

    @Override
    public void deleteArticle(int articleId) {
        entityManager.remove(getArticleById(articleId));
    }

    @Override
    public boolean articleExists(String title, String category) {
        String hql = "FROM Article as atcl WHERE atcl.title = ? and atcl.category = ?";
        int count = entityManager.createQuery(hql).setParameter(0, title)
                .setParameter(1, category).getResultList().size();
        return count > 0;
    }
}
```

## 业务Service

编写业务逻辑Service类，注入DAO类：

```java
@Service
public class ArticleService {

    @Resource
    private IArticleDAO articleDAO;

    public Article getArticleById(int articleId) {
        Article obj = articleDAO.getArticleById(articleId);
        return obj;
    }

    public List<Article> getAllArticles() {
        return articleDAO.getAllArticles();
    }

    public synchronized boolean addArticle(Article article) {
        if (articleDAO.articleExists(article.getTitle(), article.getCategory())) {
            return false;
        } else {
            articleDAO.addArticle(article);
            return true;
        }
    }

    public void updateArticle(Article article) {
        articleDAO.updateArticle(article);
    }

    public void deleteArticle(int articleId) {
        articleDAO.deleteArticle(articleId);
    }
}
```

## 编写测试用例

OK，一切写完后，就来编写我们的测试用例：

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {
    private static final Logger log = LoggerFactory.getLogger(ApplicationTests.class);

    @Resource
    private ArticleService articleService;

    /**
     * 测试增删改查
     */
    @Test
    public void test() {
        Article article = articleService.getArticleById(1);
        assertThat(article.getTitle(), is("Java Concurrency"));

        List<Article> list = articleService.getAllArticles();
        assertThat(list, notNullValue());
        assertThat(list.size(), is(3));

        boolean flag = articleService.addArticle(article);
        assertThat(flag, is(false));

        article.setTitle("Python Concurrency");
        articleService.updateArticle(article);
        Article article1 = articleService.getArticleById(1);
        assertThat(article1.getTitle(), is("Python Concurrency"));


        articleService.deleteArticle(1);
        Article article2 = articleService.getArticleById(1);
        assertThat(article2, nullValue());
    }
}
```

上面先查询id=1的文章，看看标题是否为`Java Concurrency`，然后查询所有文章列表，之后更新文章标题后再次查询看看是否更新成功。
最后删除这篇文章，再次查询结果为null。

运行结果green bar，测试通过。

更多Hibernate相关文档，请参考 [官网](http://hibernate.org/) 。

## GitHub源码

[springboot-hibernate](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-hibernate)

