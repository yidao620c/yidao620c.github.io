---
title: SpringBoot系列 - 使用RestTemplate
date: 2017-07-06 18:29:52 +0800
toc: true
categories: [ java ]
tags: [ spring ]
---

spring框架提供的RestTemplate类可用于在应用中调用rest服务，它简化了与http服务的通信方式，统一了RESTful的标准，封装了http链接，
我们只需要传入url及返回值类型即可。相较于之前常用的HttpClient，RestTemplate是一种更优雅的调用RESTful服务的方式。

RestTemplate默认依赖JDK提供http连接的能力（HttpURLConnection），如果有需要的话也可以通过setRequestFactory方法替换为例如
Apache HttpComponents、Netty或OkHttp等其它HTTP library。

本篇介绍如何使用RestTemplate，以及在SpringBoot里面的配置和注入。
<!-- more -->

## 实现逻辑

RestTemplate包含以下几个部分：

* HttpMessageConverter 对象转换器
* ClientHttpRequestFactory 默认是JDK的HttpURLConnection
* ResponseErrorHandler 异常处理
* ClientHttpRequestInterceptor 请求拦截器

用一张图可以很直观的理解：

![](https://xnstatic-1253397658.file.myqcloud.com/rest01.png)

直接使用方式很简单：

```java
public class RestTemplateTest {
	public static void main(String[] args) {
		RestTemplate restT = new RestTemplate();
		//通过Jackson JSON processing library直接将返回值绑定到对象
		Quote quote = restT.getForObject("http://gturnquist-quoters.cfapps.io/api/random", Quote.class);
		String quoteString = restT.getForObject("http://gturnquist-quoters.cfapps.io/api/random", String.class);
		System.out.println(quoteString);
	}
}
```

## 发送GET请求

```java
// 1-getForObject()
User user1 = this.restTemplate.getForObject(uri, User.class);

// 2-getForEntity()
ResponseEntity<User> responseEntity1 = this.restTemplate.getForEntity(uri, User.class);
HttpStatus statusCode = responseEntity1.getStatusCode();
HttpHeaders header = responseEntity1.getHeaders();
User user2 = responseEntity1.getBody();
  
// 3-exchange()
RequestEntity requestEntity = RequestEntity.get(new URI(uri)).build();
ResponseEntity<User> responseEntity2 = this.restTemplate.exchange(requestEntity, User.class);
User user3 = responseEntity2.getBody();
```

## 发送POST请求

```java
// 1-postForObject()
User user1 = this.restTemplate.postForObject(uri, user, User.class);

// 2-postForEntity()
ResponseEntity<User> responseEntity1 = this.restTemplate.postForEntity(uri, user, User.class);

// 3-exchange()
RequestEntity<User> requestEntity = RequestEntity.post(new URI(uri)).body(user);
ResponseEntity<User> responseEntity2 = this.restTemplate.exchange(requestEntity, User.class);
```

## 设置HTTP Header

```java
// 1-Content-Type
RequestEntity<User> requestEntity = RequestEntity
        .post(new URI(uri))
        .contentType(MediaType.APPLICATION_JSON)
        .body(user);

// 2-Accept
RequestEntity<User> requestEntity = RequestEntity
        .post(new URI(uri))
        .accept(MediaType.APPLICATION_JSON)
        .body(user);

// 3-Other
RequestEntity<User> requestEntity = RequestEntity
        .post(new URI(uri))
        .header("Authorization", "Basic " + base64Credentials)
        .body(user);
```

## 捕获异常

捕获`HttpServerErrorException`

```java
try {
    responseEntity = restTemplate.exchange(requestEntity, String.class);
} catch (HttpServerErrorException e) {
    // log error
}
```

## 自定义异常处理器

```java
public class CustomErrorHandler extends DefaultResponseErrorHandler {
    @Override
    public void handleError(ClientHttpResponse response) throws IOException {
        // todo
    }
}
```

然后设置下异常处理器：

```java
@Configuration
public class RestClientConfig {
    @Bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setErrorHandler(new CustomErrorHandler());
        return restTemplate;
    }
}
```

## 配置类

创建`HttpClientConfig`类，设置连接池大小、超时时间、重试机制等。配置如下：

```java
/**
 * - Supports both HTTP and HTTPS
 * - Uses a connection pool to re-use connections and save overhead of creating connections.
 * - Has a custom connection keep-alive strategy (to apply a default keep-alive if one isn't specified)
 * - Starts an idle connection monitor to continuously clean up stale connections.
 *
 * @author XiongNeng
 * @version 1.0
 * @since 2018/7/5
 */
@Configuration
@EnableScheduling
public class HttpClientConfig {

    private static final Logger LOGGER = LoggerFactory.getLogger(HttpClientConfig.class);

    @Resource
    private HttpClientProperties p;

    @Bean
    public PoolingHttpClientConnectionManager poolingConnectionManager() {
        SSLContextBuilder builder = new SSLContextBuilder();
        try {
            builder.loadTrustMaterial(null, new TrustStrategy() {
                public boolean isTrusted(X509Certificate[] arg0, String arg1) {
                    return true;
                }
            });
        } catch (NoSuchAlgorithmException | KeyStoreException e) {
            LOGGER.error("Pooling Connection Manager Initialisation failure because of " + e.getMessage(), e);
        }

        SSLConnectionSocketFactory sslsf = null;
        try {
            sslsf = new SSLConnectionSocketFactory(builder.build());
        } catch (KeyManagementException | NoSuchAlgorithmException e) {
            LOGGER.error("Pooling Connection Manager Initialisation failure because of " + e.getMessage(), e);
        }

        Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder
                .<ConnectionSocketFactory>create()
                .register("https", sslsf)
                .register("http", new PlainConnectionSocketFactory())
                .build();

        PoolingHttpClientConnectionManager poolingConnectionManager = new PoolingHttpClientConnectionManager(socketFactoryRegistry);
        poolingConnectionManager.setMaxTotal(p.getMaxTotalConnections());  //最大连接数
        poolingConnectionManager.setDefaultMaxPerRoute(p.getDefaultMaxPerRoute());  //同路由并发数
        return poolingConnectionManager;
    }

    @Bean
    public ConnectionKeepAliveStrategy connectionKeepAliveStrategy() {
        return new ConnectionKeepAliveStrategy() {
            @Override
            public long getKeepAliveDuration(HttpResponse response, HttpContext httpContext) {
                HeaderElementIterator it = new BasicHeaderElementIterator
                        (response.headerIterator(HTTP.CONN_KEEP_ALIVE));
                while (it.hasNext()) {
                    HeaderElement he = it.nextElement();
                    String param = he.getName();
                    String value = he.getValue();
                    if (value != null && param.equalsIgnoreCase("timeout")) {
                        return Long.parseLong(value) * 1000;
                    }
                }
                return p.getDefaultKeepAliveTimeMillis();
            }
        };
    }

    @Bean
    public CloseableHttpClient httpClient() {
        RequestConfig requestConfig = RequestConfig.custom()
                .setConnectionRequestTimeout(p.getRequestTimeout())
                .setConnectTimeout(p.getConnectTimeout())
                .setSocketTimeout(p.getSocketTimeout()).build();

        return HttpClients.custom()
                .setDefaultRequestConfig(requestConfig)
                .setConnectionManager(poolingConnectionManager())
                .setKeepAliveStrategy(connectionKeepAliveStrategy())
                .setRetryHandler(new DefaultHttpRequestRetryHandler(3, true))  // 重试次数
                .build();
    }

    @Bean
    public Runnable idleConnectionMonitor(final PoolingHttpClientConnectionManager connectionManager) {
        return new Runnable() {
            @Override
            @Scheduled(fixedDelay = 10000)
            public void run() {
                try {
                    if (connectionManager != null) {
                        LOGGER.trace("run IdleConnectionMonitor - Closing expired and idle connections...");
                        connectionManager.closeExpiredConnections();
                        connectionManager.closeIdleConnections(p.getCloseIdleConnectionWaitTimeSecs(), TimeUnit.SECONDS);
                    } else {
                        LOGGER.trace("run IdleConnectionMonitor - Http Client Connection manager is not initialised");
                    }
                } catch (Exception e) {
                    LOGGER.error("run IdleConnectionMonitor - Exception occurred. msg={}, e={}", e.getMessage(), e);
                }
            }
        };
    }

    @Bean
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setThreadNamePrefix("poolScheduler");
        scheduler.setPoolSize(50);
        return scheduler;
    }
}
```

然后再配置RestTemplateConfig类，使用之前配置好的CloseableHttpClient类注入，同时配置一些默认的消息转换器：

```java
/**
 * RestTemplate客户端连接池配置
 *
 * @author XiongNeng
 * @version 1.0
 * @since 2018/1/24
 */
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class RestTemplateConfig {

    @Resource
    private CloseableHttpClient httpClient;

    @Bean
    public RestTemplate restTemplate(MappingJackson2HttpMessageConverter jackson2HttpMessageConverter) {
        RestTemplate restTemplate = new RestTemplate(clientHttpRequestFactory());

        List<HttpMessageConverter<?>> messageConverters = new ArrayList<>();
        StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter(Charset.forName("utf-8"));
        messageConverters.add(stringHttpMessageConverter);
        messageConverters.add(jackson2HttpMessageConverter);
        restTemplate.setMessageConverters(messageConverters);

        return restTemplate;
    }

    @Bean
    public HttpComponentsClientHttpRequestFactory clientHttpRequestFactory() {
        HttpComponentsClientHttpRequestFactory rf = new HttpComponentsClientHttpRequestFactory();
        rf.setHttpClient(httpClient);
        return rf;
    }

}
```

注意，如果没有apache的HttpClient类，需要在pom文件中添加：

```xml

<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.3</version>
</dependency>
```

## 发送文件

```java
MultiValueMap<String, Object> multiPartBody = new LinkedMultiValueMap<>();
multiPartBody.add("file", new ClassPathResource("/tmp/user.txt"));
RequestEntity<MultiValueMap<String, Object>> requestEntity = RequestEntity
        .post(uri)
        .contentType(MediaType.MULTIPART_FORM_DATA)
        .body(multiPartBody);
```

## 下载文件

```java
// 小文件
RequestEntity requestEntity = RequestEntity.get(uri).build();
ResponseEntity<byte[]> responseEntity = restTemplate.exchange(requestEntity, byte[].class);
byte[] downloadContent = responseEntity.getBody();
  
// 大文件  
ResponseExtractor<ResponseEntity<File>> responseExtractor = new ResponseExtractor<ResponseEntity<File>>() {
    @Override
    public ResponseEntity<File> extractData(ClientHttpResponse response) throws IOException {
        File rcvFile = File.createTempFile("rcvFile", "zip");
        FileCopyUtils.copy(response.getBody(), new FileOutputStream(rcvFile));
        return ResponseEntity.status(response.getStatusCode()).headers(response.getHeaders()).body(rcvFile);
    }
};
File getFile = this.restTemplate.execute(targetUri, HttpMethod.GET, null, responseExtractor);
```

## Service注入

```java
@Service
public class DeviceService {
    private static final Logger logger = LoggerFactory.getLogger(DeviceService.class);

    @Resource
    private RestTemplate restTemplate;
}
```

## 实际使用例子

```java
// 开始推送消息
logger.info("解绑成功后推送消息给对应的POS机");
LoginParam loginParam = new LoginParam();
loginParam.setUsername(managerInfo.getUsername());
loginParam.setPassword(managerInfo.getPassword());
HttpBaseResponse r = restTemplate.postForObject(
        p.getPosapiUrlPrefix() + "/notifyLogin", loginParam, HttpBaseResponse.class);
if (r.isSuccess()) {
    logger.info("推送消息登录认证成功");
    String token = (String) r.getData();
    UnbindParam unbindParam = new UnbindParam();
    unbindParam.setImei(pos.getImei());
    unbindParam.setLocation(location);
    // 设置HTTP Header信息
    URI uri;
    try {
        uri = new URI(p.getPosapiUrlPrefix() + "/notify/unbind");
    } catch (URISyntaxException e) {
        logger.error("URI构建失败", e);
        return 1;
    }
    RequestEntity<UnbindParam> requestEntity = RequestEntity
            .post(uri)
            .contentType(MediaType.APPLICATION_JSON)
            .accept(MediaType.APPLICATION_JSON)
            .header("Authorization", token)
            .body(unbindParam);
    ResponseEntity<HttpBaseResponse> responseEntity = restTemplate.exchange(requestEntity, HttpBaseResponse.class);
    HttpBaseResponse r2 = responseEntity.getBody();
    if (r2.isSuccess()) {
        logger.info("推送消息解绑网点成功");
    } else {
        logger.error("推送消息解绑网点失败，errmsg = " + r2.getMsg());
    }
} else {
    logger.error("推送消息登录认证失败");
}
```

## GitHub源码

[springboot-resttemplate](https://github.com/yidao620c/SpringBootBucket/tree/master/springboot-resttemplate)

