---
title: SpringBoot系列 - 集成MongoDB
date: 2017-07-04 12:25:38 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

MongoDB是一个高性能、开源、无模式的文档型数据库，是当前NoSql数据库中比较热门的一种。
适合对大量或者无固定格式的数据进行存储，比如：日志、缓存等。对事物支持较弱，不适用复杂的多文档（多表）的级联查询。

MongoDB的适用场景：

1. 在应用服务器的日志记录
2. 存储一些监控数据
3. 应用不需要事务及复杂 join 支持
4. 应用需要2000-3000以上的读写QPS
5. 应用需要TB甚至 PB 级别数据存储
6. 应用发展迅速，需要能快速水平扩展
7. 应用要求存储的数据不丢失
8. 应用需要99.999%高可用
9. 应用需要大量的地理位置查询、文本查询

本篇将介绍如何在SpringBoot中使用MongoDB数据库。
<!-- more -->

## maven依赖

只需要添加：

```xml

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

## 安装MongoDB

MongoDB的安装教程请参考 [官网安装](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/)

我是在centos7上面安装的MongoDB 4.0，并且修改配置文件`/etc/mongod.conf`，将绑定端口设置成`0.0.0.0`，让其他主机也能访问。

启动mongodb后，添加一个用户，步骤如下：

```
mongo --port 27017
use test
db.createUser(
   {
     user: "xiongneng",
     pwd: "123456",
     roles: [ { role: "readWrite", db: "test" } ]
   }
 )
```

## 修改配置文件

修改`application.yml`配置，增加mongodb的配置项：

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://xiongneng:123456@localhost:27017/test
```

## 定义实体类

这里只是为了测试，我就定义一个简单的客户类：

```java
@Document(collection = "customer")
public class Customer {
    @Id
    private String id;
    private String firstName;
    private String lastName;

    public Customer() {
    }

    public Customer(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }

    // 省略get、set方法

    @Override
    public String toString() {
        return String.format(
                "Customer[id=%s, firstName='%s', lastName='%s']",
                id, firstName, lastName);
    }
}
```

其中的id属性是MongoDB用来唯一识别客户的主键，其实这里可以不用注解，因为默认就叫id。

另外MongoDB里面的数据会存储到集合中，Spring Data MongoDB会将Customer类的数据映射到名字叫`customer`的集合中。

## 定义DAO类

这里可继承`MongoRepository`，这样一些通用操作的方法，比如增删改查你就不用写了，只需要添加你自己自定义的其他方法：

```java
public interface CustomerRepository extends MongoRepository<Customer, String> {

    Customer findByFirstName(String firstName);

    List<Customer> findByLastName(String lastName);

}
```

这里我定义了两个简单查询方法，一个是根据名称查找单个客户，一个是根据姓来查找客户列表。
你无需去写实现类，运行时SpringBoot会自动帮你生成实现，这也是它的强大之处，将你从代码樊笼中解脱出来。

## 编写服务类

接下来编写核心的业务服务类 `CustomerService`

```java
@Service
public class CustomerService {
    @Resource
    private CustomerRepository repository;

    /**
     * 删除所有的客户
     */
    public void deleteAll() {
        repository.deleteAll();
    }

    /**
     * 保存客户
     * @param customer 客户
     */
    public void save(Customer customer) {
        repository.save(customer);
    }

    /**
     * 查询所有客户列表
     * @return 客户列表
     */
    public Iterable<Customer> findAll() {
        return repository.findAll();
    }

    /**
     * 通过名查找某个客户
     * @param firstName
     * @return
     */
    public Customer findByFirstName(String firstName) {
        return repository.findByFirstName(firstName);
    }

    /**
     * 通过姓查找客户列表
     * @param lastName
     * @return
     */
    public List<Customer> findByLastName(String lastName) {
        return repository.findByLastName(lastName);
    }
}
```

## 编写测试用例

OK，一切准备妥当之后，我们来编写测试用例：

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {
    private static final Logger log = LoggerFactory.getLogger(ApplicationTests.class);

    @Resource
    private CustomerService service;

    /**
     * 测试增删改查
     */
    @Test
    public void test() {

        service.deleteAll();

        // save a couple of customers
        service.save(new Customer("Alice", "Smith"));
        service.save(new Customer("Bob", "Smith"));

        // fetch all customers
        System.out.println("Customers found with findAll():");
        System.out.println("-------------------------------");
        int count = 0;
        for (Customer customer : service.findAll()) {
            System.out.println(customer);
            count++;
        }
        assertThat(count, is(2));

        // fetch an individual customer
        System.out.println("Customer found with findByFirstName('Alice'):");
        System.out.println("--------------------------------");
        Customer c = service.findByFirstName("Alice");
        assertThat(c, notNullValue());
        assertThat(c.getFirstName(), is("Alice"));

        System.out.println("Customers found with findByLastName('Smith'):");
        System.out.println("--------------------------------");

        List<Customer> list = service.findByLastName("Smith");
        assertThat(list, notNullValue());
        assertThat(list.size(), greaterThan(1));
        assertThat(list.size(), is(2));
    }
}
```

在上面的测试中，我先删掉数据库中所有客户数据，然后新增两条数据，查询所有的数据，根据姓或名查询。

运行测试结果为green bar，同时输出：

```
Customers found with findAll():
-------------------------------
Customer[id=5a9a30e616afd63f48afeea6, firstName='Alice', lastName='Smith']
Customer[id=5a9a30e616afd63f48afeea7, firstName='Bob', lastName='Smith']
Customer found with findByFirstName('Alice'):
--------------------------------
Customers found with findByLastName('Smith'):
```

更多关于MongoDB的使用，请移步 [官网](https://www.mongodb.com)

## GitHub源码

[springboot-mongodb](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-mongodb)

