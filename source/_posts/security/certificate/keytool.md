---
title: 【安全贴士】证书操作工具Keytool
date: 2020-05-21 03:12:22 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 数字证书
  - 安全贴士
---

生成自签名证书命令
```
keytool -genkey -v -alias oauth2 -keyalg RSA -validity 3650 -keystore server.jks
```

导出证书命令
```
keytool -exportcert -rfc -alias oauth2 -file server.crt -keystore server.jks -storepass 123456
```
<!--more-->

## 操作PKCS12格式的证书

使用keytool工具查看PKCS12格式证书
```
keytool -list -v -keystore root.p12 -storepass ${root_p12_password}
```

从PKCS12证书库中导入证书
```
keytool -import -noprompt -trustcacerts -alias caroot -file root.crt -keystore root.p12 -storetype PKCS12 -storepass ${root_p12_password}
```

从PKCS12信任证书库中删除证书
```
keytool -delete -alias xxx -keystore root.p12 -storepass ${root_p12_password}
```

## JKS和PKCS12格式转换

从PKCS12文件导出JKS证书

```bash
# 整个证书库导出
keytool -importkeystore -srckeystore certs/server.p12 -srcstoretype pkcs12 -srcstorepass ${server_p12_password} \
-destkeystore certs/server.jks -deststoretype jks -deststorepass ${server_jks_password}
```

从JKS文件导出PKCS12证书

```bash
# 整个证书库导出
keytool -importkeystore -srckeystore certs/server.jks -srcstoretype JKS -srcstorepass ${server_p12_password} \
-destkeystore certs/server.p12 -deststoretype PKCS12 -deststorepass ${server_jks_password}
```

