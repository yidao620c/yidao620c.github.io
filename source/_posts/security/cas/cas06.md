---
title: CAS教程 - Ticket存储和验证
date: 2019-01-13 11:19:22 +0800
toc: true
categories: [网络安全]
tags: [ CAS ]
---

默认情况下，cas是将票据信息存储到内存中，我们可以将票据存储到redis服务器中，cas采用的spring data redis 来控制redis

将票据存储到redis需要两个步骤：

1. 配置cas关于redis的依赖
2. 配置application.properties，添加redis的配置信息

接下来讲解如何通过Redis存储Ticket，以及通过Restful API方式验证Ticket。
<!-- more -->

## maven配置

```xml
<!--redis支持，将票据存在redis，默认是内存中-->
<dependency>
    <groupId>org.apereo.cas</groupId>
    <artifactId>cas-server-support-redis-ticket-registry</artifactId>
    <version>${cas.version}</version>
</dependency>
```

## redis配置

```yaml
cas:
  ticket:
    st:
      timeToKillInSeconds: 20 #ST票据生命周期
    pt:
      timeToKillInSeconds: 20 #PT票据生命周期
    registry:
      redis:
        host: 127.0.0.1
        database: 1
        port: 6379
        password: test
        timeout: 2000
        useSsl: false
        usePool: false
  tgc:
    crypto:
      enabled: false #表示不对TGC进行加密，默认是true，TGC其实就是TGT经过JWT加密后的值而已
```

## Ticket接口

cas可供调用的接口列表如下，通过这些接口可实现票据操作。

### 获取TGT

```java
String getTicketGrantingTicket(String Server, String username, String password);
```

* 参数server为CAS Server的访问URL；
* 参数username为登录用户名；
* 参数password为验证用的密码；

返回：验证通过则返回TGT的值，否则抛出异常；

示例：

```java
String tgt = casService.getTicketGrantingTicket("https://cas.hisign.com.cn:8443/cas", "casuser", "Mellon");
```

### 根据TGT获取ST

```java
String getServiceTicket(String Server, String ticketGrantingTicket, String service);
```

* 参数server为CAS Server的访问URL；
* 参数ticketGrantingTicket为已获得的TGT；
* 参数service为欲访问的service的URL；

返回：验证通过则返回ST的值，否则抛出异常；

示例：

```java
String st = casService.getTicketGrantingTicket("https://cas.hisign.com.cn:8443/cas", "TGT-2-6eT-cas01.example.org", "https://app.xx.com:8443/app1");
```

### 判别ST是否有效

```java
String verifySeviceTicket(String server, String serviceTicket, String service);
```

* 参数server为CAS Server的访问URL；
* 参数serviceTicket为已获得的ST；
* 参数service为欲访问的service的URL；

返回：ST有效返回登录用户名，无效返回null，若出错抛出异常；

示例：

```java
boolean String = casService.verifyServiceTicket("https://cas.hisign.com.cn:8443/cas", "ST-2-5kE-cas01.example.org", "https://app.xx.com:8443/app1");
```

### 删除TGT(相当于在CAS Server端注销)

```java
boolean deleteTicketGrantingTicket(String Server, String ticketGrantingTicket);
```

* 参数server为CAS Server的访问URL；
* 参数ticketGrantingTicket为已获得的TGT；

返回：成功返回true，否则抛出异常；

示例：

```java
boolean bool = casService.deleteTicketGrantingTicket("https://cas.hisign.com.cn:8443/cas", "TGT-2-6eT-cas01.example.org");
```

**注意事项**：参数server必须真实有效，可从配置文件获取；而参数service可以是虚构的、符合规范的格式。

### Restful接口测试

添加maven依赖：

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.3</version>
</dependency>

<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.10.0</version>
</dependency>
```

编写API接口访问工具类：

```java
/**
 * 首先客户端提交用户名、密码、及Service三个参数，
 * 如果验证成功便返回用户的TGT(Ticket Granting Ticket)至客户端,
 * 然后客户端再根据 TGT 获取用户的 ST(Service Ticket)来进行验证登录。
 * 故名思意，TGT是用于生成一个新的Ticket(ST)的Ticket，
 * 而ST则是提供给客户端用于登录的Ticket，两者最大的区别在于，
 * TGT是用户名密码验证成功之后所生成的Ticket，并且会保存在Server中及Cookie中，
 * 而ST则必须是是根据TGT来生成，主要用于登录，并且当登录成功之后 ST 则会失效。
 */
public class CasServerUtil {

    // 登录服务器地址
    private static final String CAS_SERVER_PATH = "https://cas.server.com:8443";

    // 登录地址的token
    private static final String GET_TOKEN_URL = CAS_SERVER_PATH + "/v1/tickets";


    public static void main(String[] args) {

        try {
            String tgt = getTGT("tingfeng", "tingfeng");
            System.out.println("TGT：" + tgt);

            String service = "http://app1.com:8181/fire/users.html";
            String st = getST(tgt, service);
            System.out.println("ST：" + st);

            System.out.println(service + "?ticket=" + st);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    /**
     * 获取TGT
     */
    public static String getTGT(String username, String password) {
        try{
            CookieStore httpCookieStore = new BasicCookieStore();
//        CloseableHttpClient client = createHttpClientWithNoSsl(httpCookieStore);

            CloseableHttpClient client = HttpClients.createDefault();

            HttpPost httpPost = new HttpPost(GET_TOKEN_URL);
            List<NameValuePair> params = new ArrayList<NameValuePair>();
            params.add(new BasicNameValuePair("username", username));
            params.add(new BasicNameValuePair("password", password));
            httpPost.setEntity(new UrlEncodedFormEntity(params));
            HttpResponse response = client.execute(httpPost);

//        System.out.println("\n 获取TGT，Header响应");
//        Header[] allHeaders = response.getAllHeaders();
//        for (int i = 0; i < allHeaders.length; i++) {
//            System.out.println("Key：" + allHeaders[i].getName() + "，Value：" + allHeaders[i].getValue() + "，Elements:" + Arrays.toString(allHeaders[i].getElements()));
//        }

            Header headerLocation = response.getFirstHeader("Location");
            String location = headerLocation == null ? null : headerLocation.getValue();

            System.out.println("Location：" + location);

            if (location != null) {
                return location.substring(location.lastIndexOf("/") + 1);
            }
        }catch (Exception e){
            e.printStackTrace();
        }
//
        return null;
    }


    /**
     * 获取ST
     */
    public static String getST(String TGT, String service){
        try {

//        CookieStore httpCookieStore = new BasicCookieStore();
//        CloseableHttpClient client = createHttpClientWithNoSsl(httpCookieStore);

            CloseableHttpClient client = HttpClients.createDefault();


            // service 需要encoder编码
            // service = URLEncoder.encode(service, "utf-8");

            HttpPost httpPost = new HttpPost(GET_TOKEN_URL + "/" + TGT);
            List<NameValuePair> params = new ArrayList<NameValuePair>();
            params.add(new BasicNameValuePair("service", service));
            httpPost.setEntity(new UrlEncodedFormEntity(params));
            HttpResponse response = client.execute(httpPost);

//        System.out.println("\n 获取ST，Header响应");
//        Header[] allHeaders = response.getAllHeaders();
//        for (int i = 0; i < allHeaders.length; i++) {
//            System.out.println("Key：" + allHeaders[i].getName() + "，Value：" + allHeaders[i].getValue() + "，Elements:" + Arrays.toString(allHeaders[i].getElements()));
//        }
//
//
//        List<Cookie> cookies = httpCookieStore.getCookies();
//        System.out.println("Cookie.size：" + cookies.size());
//        for (Cookie cookie : cookies) {
//            System.out.println("Cookie: " + new Gson().toJson(cookie));
//        }

            String st = readResponse(response);
            return st == null ? null : (st == "" ? null : st);
        }catch (Exception e){
            e.printStackTrace();
        }
        return null;
    }


    /**
     * 读取 response body 内容为字符串
     *
     * @param response
     * @return
     * @throws IOException
     */
    private static String readResponse(HttpResponse response) throws IOException {
        BufferedReader in = new BufferedReader(new InputStreamReader(response.getEntity().getContent()));
        String result = new String();
        String line;
        while ((line = in.readLine()) != null) {
            result += line;
        }
        return result;
    }


    /**
     * 创建模拟客户端（针对 https 客户端禁用 SSL 验证）
     *
     * @param cookieStore 缓存的 Cookies 信息
     * @return
     * @throws Exception
     */
    private static CloseableHttpClient createHttpClientWithNoSsl(CookieStore cookieStore) throws Exception {
        // Create a trust manager that does not validate certificate chains
        TrustManager[] trustAllCerts = new TrustManager[]{
                new X509TrustManager() {
                    @Override
                    public X509Certificate[] getAcceptedIssuers() {
                        return null;
                    }

                    @Override
                    public void checkClientTrusted(X509Certificate[] certs, String authType) {
                        // don't check
                    }

                    @Override
                    public void checkServerTrusted(X509Certificate[] certs, String authType) {
                        // don't check
                    }
                }
        };

        SSLContext ctx = SSLContext.getInstance("TLS");
        ctx.init(null, trustAllCerts, null);
        LayeredConnectionSocketFactory sslSocketFactory = new SSLConnectionSocketFactory(ctx);
        return HttpClients.custom()
                .setSSLSocketFactory(sslSocketFactory)
                .setDefaultCookieStore(cookieStore == null ? new BasicCookieStore() : cookieStore)
                .build();
    }
}
```


