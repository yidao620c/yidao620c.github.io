---
title: 【安全贴士】Java实现证书双向认证
date: 2020-05-26 19:25:22 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 数字证书
  - 安全贴士
---

SSL/TLS 证书是用于用户浏览器和网站服务器之间的数据传输加密，实现互联网传输安全保护，大多数情况下指的是服务器证书。
服务器证书是用于向浏览器客户端验证服务器，这种是属于单向认证的SSL证书。但是，如果服务器需要对客户端进行身份验证，
该怎么办？这就需要双向认证证书。

为什么需要另一种认证方式的证书？因为当同时使用两种认证方式的证书时，有助于双方（即客户端和服务器端）之间的相互认证。
另外，与标准SSL证书不同的是，双向认证的SSL证书实际上被称作为个人认证证书(PAC)。

双向认证流程图如下：
![img.png](https://xnstatic-1253397658.file.myqcloud.com/20230716-01.png)

1. 客户端发起建立HTTPS连接请求，将SSL协议版本的信息发送给服务端；
2. 服务器端将本机的公钥证书（server.crt）发送给客户端；
3. 客户端读取公钥证书（server.crt），取出了服务端公钥；
4. 客户端将客户端公钥证书（client.crt）发送给服务器端；
5. 服务器端使用根证书（root.crt）解密客户端公钥证书，拿到客户端公钥；
6. 客户端发送自己支持的加密方案给服务器端；
7. 服务器端根据自己和客户端的能力，选择一个双方都能接受的加密方案，使用客户端的公钥加密后发送给客户端；
8. 客户端使用自己的私钥解密加密方案，生成一个随机数R，使用服务器公钥加密后传给服务器端；
9. 服务端用自己的私钥去解密这个密文，得到了密钥R
10. 服务端和客户端在后续通讯过程中就使用这个密钥R进行通信了。

本篇直接通过代码演示如何在Java实现证书的双向认证。

<!--more-->

## HttpClientProperties类

``` java
@Component
@ConfigurationProperties(prefix = "com.xncoding.https.httpclient")
public class HttpClientProperties {
    /**
     * 是否开启服务端证书校验
     */
    private boolean enabled = true;
    /**
     * 是否开启客户端证书发送
     */
    private boolean clientCert = false;
    /**
     * 证书格式：PKCS12/JKS
     */
    private String keystoreType = "PKCS12";
    /**
     * CA根证书密钥库文件
     */
    private String caRootCertKeyStore;
    /**
     * CA根证书密钥库密码
     */
    private String caRootCertPassword;
    /**
     * 客户端证书库文件
     */
    private String clientCertKeyStore;
    /**
     * 客户端证书库密码
     */
    private String clientCertPassword;
    /**
     * 建立连接的超时时间
     */
    private int connectTimeout = 20000;
    /**
     * 连接不够用的等待时间
     */
    private int requestTimeout = 20000;
    /**
     * 每次请求等待返回的超时时间
     */
    private int socketTimeout = 30000;
    /**
     * 同路由最大连接数
     */
    private int defaultMaxPerRoute = 100;
    /**
     * 连接池最大连接数
     */
    private int maxTotalConnections = 300;
    /**
     * 连接保持活跃的时间Keep-Alive（毫秒）
     */
    private int defaultKeepAliveTimeMillis = 20000;
    /**
     * 空闲连接的生存时间（秒）
     */
    private int closeIdleConnectionWaitTimeSecs = 30;
}
```

## X509Util工具类

``` java
public class X509Util {

    private static final Logger logger = LoggerFactory.getLogger(X509Util.class);

    public static SSLContext initSslContext(HttpClientProperties properties) {
        // 加载服务端信任Keystore
        X509TrustManager origTrustmanager = getX509TrustManager(properties);
        TrustManager[] wrappedTrustManagers = new TrustManager[]{
            new X509TrustManager() {
                @Override
                public X509Certificate[] getAcceptedIssuers() {
                    logger.debug(">>>>>>>>>>>>>> getAcceptedIssuers 00000000000000000 start ...");
                    return origTrustmanager.getAcceptedIssuers();
                }

                @Override
                public void checkClientTrusted(X509Certificate[] certs, String authType) throws CertificateException {
                    logger.debug(">>>>>>>>>>>>>> checkClientTrusted 111111111111 start ...");
                    origTrustmanager.checkClientTrusted(certs, authType);
                }

                @Override
                public void checkServerTrusted(X509Certificate[] certs, String authType) throws CertificateException {
                    logger.debug(">>>>>>>>>>>>>> checkServerTrusted 222222222222222 start ...");
                    // 校验逻辑：如果证书链第一个证书在信任库，则不会去校验该证书过期时间。其他的都会校验
                    origTrustmanager.checkServerTrusted(certs, authType);
                    // 过期时间校验
                    for (X509Certificate cert : certs) {
                        cert.checkValidity();
                    }
                    if (certs.length > 1) {
                        for (int i = 1; i < certs.length; i++) {
                            X509Certificate ca = certs[i];
                            // 根证书的扩展属性中“基本限制”属性是否包括“CA”
                            if (ca.getBasicConstraints() == -1) {
                                logger.error("check CA certificate error");
                                throw new CertificateException("not a CA certificate");
                            }
                            // 根证书的扩展属性中“密钥用法”中是否包括“证书签名”属性
                            /*
                             * KeyUsage ::= BIT STRING {
                             *   digitalSignature        (0),
                             *   nonRepudiation          (1),
                             *   keyEncipherment         (2),
                             *   dataEncipherment        (3),
                             *   keyAgreement            (4),
                             *   keyCertSign             (5),
                             *   cRLSign                 (6),
                             *   encipherOnly            (7),
                             *   decipherOnly            (8) }
                             */
                            boolean[] keyUsages = ca.getKeyUsage();
                            if (!keyUsages[5]) {
                                logger.error("check keyusage keyCertSign error");
                                throw new CertificateException("keyusage has not value keyCertSign");
                            }
                        }
                    }
                }
            }
        };

        try {
            SSLContext sslContext = SSLContext.getInstance("TLSv1.2");
            if (properties.isClientCert()) { // 如果开启客户端证书校验，则需要发送客户端证书
                sslContext.init(getX509KeyManagers(properties), wrappedTrustManagers, new java.security.SecureRandom());
            } else { // 否则不需要发送客户端证书
                sslContext.init(null, wrappedTrustManagers, new java.security.SecureRandom());
            }
            return sslContext;
        } catch (NoSuchAlgorithmException | KeyManagementException e) {
            throw new RuntimeException("init sslContext error");
        }
    }

    public static X509TrustManager getX509TrustManager(HttpClientProperties properties) {
        String caRootCertKeyStore = properties.getCaRootCertKeyStore();
        try {
            PathUtils.checkRegularAndSecure(caRootCertKeyStore);
        } catch (SecurityException e) {
            throw new RuntimeException("init trustManager, check regular and secure error", e);
        }
        try (FileInputStream rootKeyStore = new FileInputStream(caRootCertKeyStore)) {
            // 加载服务端信任根证书库
            KeyStore trustKeyStore = KeyStore.getInstance(properties.getKeystoreType());
            trustKeyStore.load(rootKeyStore, properties.getCaRootCertPassword().toCharArray());
            // 初始化服务端信任证书管理器
            TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            tmf.init(trustKeyStore);
            TrustManager[] trustManagers = tmf.getTrustManagers();
            return (X509TrustManager) trustManagers[0];
        } catch (KeyStoreException | IOException | NoSuchAlgorithmException | CertificateException e) {
            throw new RuntimeException("init trustManager error", e);
        }
    }

    private static KeyManager[] getX509KeyManagers(HttpClientProperties properties) {
        String clientCertKeyStore = properties.getClientCertKeyStore();
        try {
            PathUtils.checkRegularAndSecure(clientCertKeyStore);
        } catch (SecurityException e) {
            throw new RuntimeException("init KeyManager, check regular and secure error", e);
        }
        try (FileInputStream clientKeystore = new FileInputStream(clientCertKeyStore)) {
            // 加载客户端证书库
            KeyStore clientKeyStore = KeyStore.getInstance(properties.getKeystoreType());
            clientKeyStore.load(clientKeystore, properties.getClientCertPassword().toCharArray());
            KeyManagerFactory keyManagerFactory = KeyManagerFactory
                .getInstance(KeyManagerFactory.getDefaultAlgorithm());
            keyManagerFactory.init(clientKeyStore, properties.getClientCertPassword().toCharArray());
            return keyManagerFactory.getKeyManagers();
        } catch (KeyStoreException | IOException | NoSuchAlgorithmException | CertificateException | UnrecoverableKeyException e) {
            throw new RuntimeException("init keyManagers error", e);
        }
    }
}
```

## AbstractUserInfoInterceptor

``` java
public abstract class AbstractUserInfoInterceptor implements HttpRequestInterceptor {

    private static final Logger _logger = LoggerFactory.getLogger(AbstractUserInfoInterceptor.class);

    /**
     * 内部微服务之间调用增加的用户信息头
     */
    private static final String HEADER_ACCESS_USER = "access-user";

    public void process(HttpRequest request, HttpContext context) {
        _logger.debug("ClientIPInterceptor start to handle...");
        request.addHeader(HEADER_ACCESS_USER, loadUserInfo());
    }

    /**
     * 加载用户信息
     *
     * @return 用户信息
     */
    protected abstract String loadUserInfo();
}
```

## AbstractClientIPInterceptor

``` java
public abstract class AbstractClientIPInterceptor implements HttpRequestInterceptor {

    private static final Logger _logger = LoggerFactory.getLogger(AbstractClientIPInterceptor.class);

    /**
     * 内部微服务之间调用增加的IP地址头
     */
    private static final String HEADER_X_REMOTE_USER_IP = "X-Remote-User-IP";

    public void process(HttpRequest request, HttpContext context) {
        _logger.debug("ClientIPInterceptor start to handle...");
        request.addHeader(HEADER_X_REMOTE_USER_IP, loadRemoteClientIp());
    }

    /**
     * 加载用户真实IP地址
     *
     * @return 用户真实IP地址
     */
    protected abstract String loadRemoteClientIp();
}
```

## 主配置类SecurityHttpClientConfig

``` java
/**
 * 支持HTTPS协议访问和证书双向认证的HttpClient配置类
 *
 * @since 2020-02-10
 */
@Configuration
@ConditionalOnProperty(value = "com.xncoding.https.httpclient.enabled", havingValue = "true")
@EnableScheduling
@EnableConfigurationProperties({HttpClientProperties.class})
public class SecurityHttpClientConfig {

    private static final Logger logger = LoggerFactory.getLogger(SecurityHttpClientConfig.class);

    @Autowired
    private HttpClientProperties properties;
    @Autowired
    protected ApplicationContext context;

    @Bean(name = "restTemplate")
    public RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
        logger.info("loadBalanced RestTemplate initialize");
        return restTemplateBuilder.build();
    }

    @Bean
    @DependsOn(value = {"customRestTemplateCustomizer"})
    public RestTemplateBuilder restTemplateBuilder(
        MappingJackson2HttpMessageConverter jackson2HttpMessageConverter, SSLContext sslContext) {
        RestTemplateBuilder builder = new RestTemplateBuilder(customRestTemplateCustomizer(sslContext));

        List<HttpMessageConverter<?>> messageConverters = new ArrayList<>();
        StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter(StandardCharsets.UTF_8);
        FormHttpMessageConverter formMessageConverter = new FormHttpMessageConverter();
        messageConverters.add(stringHttpMessageConverter);
        messageConverters.add(jackson2HttpMessageConverter);
        messageConverters.add(formMessageConverter);
        builder.messageConverters(messageConverters);

        return builder;
    }

    @Bean
    public RestTemplateCustomizer customRestTemplateCustomizer(SSLContext sslContext) {
        return restTemplate -> {
            HttpComponentsClientHttpRequestFactory rf = new HttpComponentsClientHttpRequestFactory();
            rf.setHttpClient(httpClient(sslContext));
            restTemplate.setRequestFactory(rf);
        };
    }

    @Bean
    public SSLContext sslContext() {
        logger.info("SecurityHttpClientConfig init sslContext...");
        return X509Util.initSslContext(properties);
    }

    @Bean
    public PoolingHttpClientConnectionManager poolingConnectionManager(SSLContext sslContext) {
        SSLConnectionSocketFactory sslsf;
        try {
            HostnameVerifier hostnameVerifier = (s, sslSession) -> {
                try {
                    Certificate[] certs = sslSession.getPeerCertificates();
                    X509Certificate x509 = (X509Certificate) certs[0];
                } catch (SSLPeerUnverifiedException e) {
                    logger.error("hostnameVerifier error", e);
                    return false;
                }
                return true;
            };
            sslsf = new SSLConnectionSocketFactory(sslContext, new String[]{"TLSv1.2"}, null, hostnameVerifier);
        } catch (Exception e) {
            logger.error("Pooling Connection Manager Initialisation failure");
            throw new RuntimeException("Pooling Connection Manager Initialisation failure", e);
        }
        Registry<ConnectionSocketFactory> socketFactoryRegistry = RegistryBuilder
            .<ConnectionSocketFactory>create()
            .register("https", sslsf)
            .register("http", new PlainConnectionSocketFactory())
            .build();

        PoolingHttpClientConnectionManager poolingConnectionManager
            = new PoolingHttpClientConnectionManager(socketFactoryRegistry);
        poolingConnectionManager.setMaxTotal(properties.getMaxTotalConnections());  //连接池最大连接数
        poolingConnectionManager.setDefaultMaxPerRoute(properties.getDefaultMaxPerRoute());  //同路由最大连接数
        return poolingConnectionManager;
    }

    @Bean
    public ConnectionKeepAliveStrategy connectionKeepAliveStrategy() {
        return (response, httpContext) -> {
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
            return properties.getDefaultKeepAliveTimeMillis();
        };
    }

    private CloseableHttpClient httpClient(SSLContext sslContext) {
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectionRequestTimeout(properties.getRequestTimeout()) //从池中获取请求的时间
            .setConnectTimeout(properties.getConnectTimeout()) //连接到服务器的时间
            .setSocketTimeout(properties.getSocketTimeout()).build(); //读取信息时间

        HttpClientBuilder httpClientBuilder = HttpClients.custom()
            .setDefaultRequestConfig(requestConfig)
            .setConnectionManager(poolingConnectionManager(sslContext))
            .setConnectionManagerShared(true)
            .setKeepAliveStrategy(connectionKeepAliveStrategy())
            .setRetryHandler(new DefaultHttpRequestRetryHandler(3, true));

        // 增加请求拦截器
        Map<String, HttpRequestInterceptor> interceptorMap = context.getBeansOfType(HttpRequestInterceptor.class);
        if (interceptorMap.size() > 0) {
            for (HttpRequestInterceptor interceptor : interceptorMap.values()) {
                httpClientBuilder.addInterceptorLast(interceptor);
            }
        }
        return httpClientBuilder.build();
    }

    /**
     * You can't set an idle connection timeout in the config for Apache HTTP Client. The reason is that there is a
     * performance overhead in doing so.
     * <properties>
     * The documentation clearly states why, and gives an example of an idle connection monitor implementation you can
     * copy. Essentially this is another thread that you run to periodically call closeIdleConnections on
     * HttpClientConnectionManager
     * <properties>
     * http://hc.apache.org/httpcomponents-client-ga/tutorial/html/connmgmt.html
     *
     * @return 线程
     */
    @Bean
    public Runnable idleConnectionMonitor() {
        return new Runnable() {
            @Override
            @Scheduled(fixedDelay = 10000)
            public void run() {
                try {
                    PoolingHttpClientConnectionManager connectionManager = poolingConnectionManager(sslContext());
                    logger.trace("run IdleConnectionMonitor - Closing expired and idle connections...");
                    connectionManager.closeExpiredConnections();
                    connectionManager
                        .closeIdleConnections(properties.getCloseIdleConnectionWaitTimeSecs(), TimeUnit.SECONDS);
                } catch (Exception e) {
                    logger.error("run IdleConnectionMonitor - Exception occurred.", e);
                }
            }
        };
    }

    @Bean
    @ConditionalOnMissingBean(TaskScheduler.class)
    public TaskScheduler taskScheduler() {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setThreadNamePrefix("poolScheduler");
        scheduler.setPoolSize(10);
        return scheduler;
    }

}
```

## 客户端使用配置

``` yml
com:
  xncoding:
    https:
      httpclient:
        enabled: true
        ca-root-cert-key-store: /cert/root.p12  #根证书库
        ca-root-cert-password: 333333 #根证书库密码
        client-cert: true #开启客户端证书
        client-cert-key-store: /cert/server.p12 #客户端证书库
        client-cert-password: 222222 #客户端证书库密码
```

然后在Service中直接注入`RestTemplate`类即可。

``` java
@Autowired
private RestTemplate restTemplate;
```

后面该咋调用还是咋调用，客户端使用者不感知。

## Tomcat证书双向认证

上面的是Java客户端的代码，双向认证客户端和服务端都要配置自己的证书和信任证书，以下是服务端Tomcat8的配置

``` xml
<Connector port="9443" protocol="org.apache.coyote.http11.Http11NioProtocol" maxThreads="150" SSLEnabled="true">
    <UpgradeProtocol className="org.apache.coyote.http2.Http2Protocol" />
    <SSLHostConfig
            truststoreFile="conf/root.p12"
            truststorePassword="222222"
            truststoreType="PKCS12"
            certificateVerification="required"
            truststoreAlgorithm="SunX509"
            sslProtocol="TLS">
        <Certificate certificateKeystoreFile="conf/server.p12"
           certificateKeystorePassword="222222"
           certificateKeystoreType="PKCS12" type="RSA" />
    </SSLHostConfig>
</Connector>
```

或者是下面的配置：

``` xml
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol" SSLEnabled ="true" 
        sslProtocol ="TLS" maxThreads="150"
        acceptCount="100" maxHttpHeaderSize="8192" maxKeepAliveRequests="100" scheme="https" secure="true"
        keystoreFile="conf/server.p12" keystorePass="333333" keystoreType="PKCS12"
        clientAuth="true" truststoreFile="root.p12" truststorePass="66666" truststoreType="PKCS12" truststoreAlgorithm="SunX509"
        ciphers="TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
        maxPostSize="10240" connectionTimeout="20000" allowTrace="false" xpoweredBy="false" 
        server="WebServer" URIEncoding="UTF-8" sslEnabledProtocols="TLSv1.2,TLSv1.3"/>
```
