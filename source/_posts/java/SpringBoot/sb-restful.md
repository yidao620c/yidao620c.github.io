---
title: SpringBoot系列 - 实现RESTful接口
date: 2017-07-05 12:10:37 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

REST，即Representational State Transfer的缩写，对这个词组的翻译是`表现层状态转化`。

RESTful是一种软件设计风格，就是目前最流行的一种互联网软件架构。它结构清晰、符合标准、易于理解、扩展方便，所以正得到越来越多网站的采用。

SpringMVC对RESTful风格的接口有着天然的支持，本篇将讲述如何在SpringBoot中怎样写。
<!-- more -->

## 几个注解

在讲述使用之前，想要理解SpringMVC的几个常用注解：

1. @Controller：修饰class，用来创建处理http请求的对象
1.

@RestController：Spring4之后加入的注解，原来在@Controller中返回json需要@ResponseBody来配合，如果直接用@RestController替代@Controller就不需要再配置@ResponseBody，默认返回json格式。

1. @RequestMapping：配置url映射
1. @PostMapping: 这个是@RequestMapping+POST方法的简写
1. @RequestHeader: 请求Header参数
1. @PathVariable: URL路径参数，比如/user/{id}中的id参数
1. @RequestParam: URL请求参数，比如/user?id=1中的id参数
1. @RequestBody: 请求Body参数

下面我们尝试使用Spring MVC来实现一组对User对象操作的RESTful API，配合注释详细说明在Spring MVC中如何映射HTTP请求、如何传参、如何编写单元测试。

## API设计

RESTful API具体设计如下：

 请求类型  | URL         | 功能说明     
-------|-------------|----------
 GET   | /users      | 查询用户列表   
 POST  | /users      | 创建一个用户   
 GET   | /users/{id} | 根据id查询用户 
 PUT   | /users/{id} | 根据id更新用户 
 DELTE | /users/{id} | 更加id删除用户 

RESTful架构有一些典型的设计误区，就是URI包含动词。因为"资源"表示一种实体，所以应该是名词，URI不应该有动词，动词应该放在HTTP协议中。
上面设计的API的URI中都是名词。

## 实体定义

```java
public class User {
    private Long id;
    private String name;
    private Integer age;
    // 下面省略getter/setter
}
```

## Controller实现

接下来就可以编写RestController了，这里为了演示，会将数据保存到内存Map中，实际使用肯定是保存到数据库中。

```java
/**
 * 接口类
 */
@RestController
@RequestMapping(value = "/users")
public class UserController {

    private static final Logger _logger = LoggerFactory.getLogger(UserController.class);

    // 创建线程安全的Map
    private static Map<Long, User> users = Collections.synchronizedMap(new HashMap<Long, User>());

    @RequestMapping(value = "/", method = RequestMethod.GET)
    public BaseResponse<List<User>> getUserList() {
        // 处理"/users/"的GET请求，用来获取用户列表
        // 还可以通过@RequestParam从页面中传递参数来进行查询条件或者翻页信息的传递
        List<User> r = new ArrayList<>(users.values());
        return new BaseResponse<>(true, "查询列表成功", r);
    }

    @RequestMapping(value = "/", method = RequestMethod.POST)
    public BaseResponse<String> postUser(@ModelAttribute User user) {
        // 处理"/users/"的POST请求，用来创建User
        // 除了@ModelAttribute绑定参数之外，还可以通过@RequestParam从页面中传递参数
        users.put(user.getId(), user);
        return new BaseResponse<>(true, "新增成功", "");
    }

    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public BaseResponse<User> getUser(@PathVariable Long id) {
        // 处理"/users/{id}"的GET请求，用来获取url中id值的User信息
        // url中的id可通过@PathVariable绑定到函数的参数中
        return new BaseResponse<>(true, "查询成功", users.get(id));
    }

    @RequestMapping(value = "/{id}", method = RequestMethod.PUT)
    public BaseResponse<String> putUser(@PathVariable Long id, @ModelAttribute User user) {
        // 处理"/users/{id}"的PUT请求，用来更新User信息
        User u = users.get(id);
        u.setName(user.getName());
        u.setAge(user.getAge());
        users.put(id, u);
        return new BaseResponse<>(true, "更新成功", "");
    }

    @RequestMapping(value = "/{id}", method = RequestMethod.DELETE)
    public BaseResponse<String> deleteUser(@PathVariable Long id) {
        // 处理"/users/{id}"的DELETE请求，用来删除User
        users.remove(id);
        return new BaseResponse<>(true, "删除成功", "");
    }

}
```

## 测试

接下来就是写一个测试来测一下，SpringMVC的内置测试支持非常方便。

先写一个json和对象之间相互转化的工具类，使用Jackson库：

```java
public class JacksonUtil {
    private static ObjectMapper mapper = new ObjectMapper();

    public static String bean2Json(Object obj) {
        try {
            return mapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
            return null;
        }
    }

    public static <T> T json2Bean(String jsonStr, TypeReference<T> typeReference) {
        try {
            return mapper.readValue(jsonStr, typeReference);
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }
}
```

然后再编写单元测试：

```java
@AutoConfigureMockMvc
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class, webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class ApplicationTests {
    @Autowired
    private MockMvc mvc;

    @Test
    public void testUserController() throws Exception {
        // 测试UserController
        RequestBuilder request;

        // 1、get查一下user列表，应该为空
        request = get("/users/");
        MvcResult result = mvc.perform(request)
                .andExpect(status().isOk())
                .andReturn();
        String content = result.getResponse().getContentAsString();
        BaseResponse<List<User>> response = JacksonUtil.json2Bean(content, new TypeReference<BaseResponse<List<User>>>() {});
        assertThat(response.isSuccess(), is(true));
        assertThat(response.getMsg(), is("查询列表成功"));
        assertThat(((List) response.getData()).size(), is(0));

        // 2、post提交一个user
        request = post("/users/")
                .param("id", "1")
                .param("name", "测试大师")
                .param("age", "20");
        result = mvc.perform(request)
                .andExpect(status().isOk())
                .andReturn();
        content = result.getResponse().getContentAsString();
        BaseResponse<String> response1 = JacksonUtil.json2Bean(content, new TypeReference<BaseResponse<String>>() {});
        assertThat(response1.isSuccess(), is(true));
        assertThat(response1.getMsg(), is("新增成功"));

        // 3、get获取user列表，应该有刚才插入的数据
        request = get("/users/");
        result = mvc.perform(request)
                .andExpect(status().isOk())
                .andReturn();
        content = result.getResponse().getContentAsString();
        BaseResponse<List<User>> response2 = JacksonUtil.json2Bean(content, new TypeReference<BaseResponse<List<User>>>() {});
        assertThat(response2.isSuccess(), is(true));
        assertThat(response2.getMsg(), is("查询列表成功"));
        assertThat((response2.getData()).size(), is(1));

        // 4、put修改id为1的user
        request = put("/users/1")
                .param("name", "测试终极大师")
                .param("age", "30");
        result = mvc.perform(request)
                .andExpect(status().isOk())
                .andReturn();
        content = result.getResponse().getContentAsString();
        BaseResponse<String> response3 = JacksonUtil.json2Bean(content, new TypeReference<BaseResponse<String>>() {});
        assertThat(response3.isSuccess(), is(true));
        assertThat(response3.getMsg(), is("更新成功"));

        // 5、get一个id为1的user
        request = get("/users/1");
        result = mvc.perform(request)
                .andExpect(status().isOk())
                .andReturn();
        content = result.getResponse().getContentAsString();
        BaseResponse<User> response4 = JacksonUtil.json2Bean(content, new TypeReference<BaseResponse<User>>() {});
        assertThat(response4.isSuccess(), is(true));
        assertThat(response4.getMsg(), is("查询成功"));
        User user = response4.getData();
        assertThat(user.getId(), is(1L));
        assertThat(user.getName(), is("测试终极大师"));

        // 6、del删除id为1的user
        request = delete("/users/1");
        result = mvc.perform(request)
                .andExpect(status().isOk())
                .andReturn();
        content = result.getResponse().getContentAsString();
        BaseResponse<String> response5 = JacksonUtil.json2Bean(content, new TypeReference<BaseResponse<String>>() {});
        assertThat(response5.isSuccess(), is(true));
        assertThat(response5.getMsg(), is("删除成功"));

        // 7、get查一下user列表，应该为空
        request = get("/users/");
        result = mvc.perform(request)
                .andExpect(status().isOk())
                .andReturn();
        content = result.getResponse().getContentAsString();
        BaseResponse<List<User>> response6 = JacksonUtil.json2Bean(content, new TypeReference<BaseResponse<List<User>>>() {});
        assertThat(response6.isSuccess(), is(true));
        assertThat(response6.getMsg(), is("查询列表成功"));
        assertThat((response6.getData()).size(), is(0));
    }

}
```

运行测试，结果为green bar，测试通过。

## GitHub源码

[springboot-restful](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-restful)

