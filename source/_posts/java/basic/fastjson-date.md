---
title: fastjson对Date的处理
date: 2015-04-22 13:45:15 +0800
toc: true
categories: [ java ]
tags: [ java ]
---

fastjson对日期的序列化方式

一种方法是通过注解

```java
@JSONField (format="yyyy-MM-dd HH:mm:ss")
public Date birthday;
```

另一种是通过SerializeConfig：

```java
private static SerializeConfig mapping = new SerializeConfig();
private static String dateFormat;
static {
    dateFormat = "yyyy-MM-dd HH:mm:ss";
    mapping.put(Date.class, new SimpleDateFormatSerializer(dateFormat));
}
```

<!-- more -->

json字符串中使用单引号：

```java
String text = JSON.toJSONString(object, SerializerFeature.UseSingleQuotes);
```

字段显示不同的key：

```java
public class User {
    @JSONField(name="ID")
    public int getId() { ... }
}

User user = ...;
JSON.toJSONString(user); // {"ID":001}
```

类的反序列化 JavaBean：

```java
String text = ...; // {"r":255,"g":0,"b":0,"alpha":255}
Color color = JSON.parseObject(text, Color.class);
```

数组：

```java
String text = ...; // [{ ... }, { ... }]
List<User> users = JSON.parseArray(text, User.class);
```

泛型：

```java
String text = ...; // {"name":{"name":"ljw",age:18} }
Map<String, User> userMap = JSON.parseObject(text, new TypeReference<Map<String, User>>() {});
```

### 自定义序列化代码示例

```java
public class JsonUtil {
    private static SerializeConfig mapping = new SerializeConfig();
    private static String dateFormat;
    static {
        dateFormat = "yyyy-MM-dd HH:mm:ss";
    }

    /**
     * 默认的处理时间
     *
     * @param jsonText
     * @return
     */
    public static String toJSON(Object jsonText) {
        return JSON.toJSONString(jsonText,
                SerializerFeature.WriteDateUseDateFormat);
    }

    /**
     * 自定义时间格式
     *
     * @param jsonText
     * @return
     */
    public static String toJSON(String dateFormat, String jsonText) {
        mapping.put(Date.class, new SimpleDateFormatSerializer(dateFormat));
        return JSON.toJSONString(jsonText, mapping);
    }
}
```

### 自定义日期格式反序列化示例

先自定义一个日期解析类：

```java
public class MyDateFormatDeserializer extends DateFormatDeserializer {

        private String myFormat;

        public MyDateFormatDeserializer(String myFormat) {
            super();
            this.myFormat = myFormat;
        }

        @Override
        protected <Date> Date cast(DefaultJSONParser parser, Type clazz, Object fieldName, Object val) {
            if (myFormat == null) {
                return null;
            }
            if (val instanceof String) {
                String strVal = (String) val;
                if (strVal.length() == 0) {
                    return null;
                }

                try {
                    return (Date) new SimpleDateFormat(myFormat).parse((String)val);
                } catch (ParseException e) {
                    throw new JSONException("parse error");
                }
            }
            throw new JSONException("parse error");
        }
    }
```

User类

```java
public class User {

    public String name;
    public int height;

    @JSONField(name = "com-google-com")
    public void setName(String name) {
        this.name = name;
    }

    @JSONField(format = "yyyy-MM/dd HH:mm:ss")
    public Date birthday;
}
```

测试如下：

```java
/**
 * @param args
 * @throws IOException
 */
public static void main(String[] args) throws IOException, ParseException {

    String json = "{\"name\":\"22323\", \"age\": 1234," +
            " \"birthday\": \"2012-12/12 12:12:12\"}";
    Test t = JSON.parseObject(json, Test.class, mapping,
            JSON.DEFAULT_PARSER_FEATURE, new Feature[0]);
    System.out.println(t.name);
    System.out.println(t.height);
    System.out.println(t.birthday);
    System.out.println(
            new SimpleDateFormat("yyyy-MM/dd HH:mm:ss").parse("2012-12/12 12:12:12"));
}
```

### 总结

对于JSONField注解，好像只对序列号的格式有影响，反序列化不管这个，不知道为什么，
只能自己写个解析类了，不过这样就更灵活了，可以在里面写很多处理逻辑，
比如json字符串里面日期格式并不是标准格式的时候，就可以先转成标准格式再去解析了。

