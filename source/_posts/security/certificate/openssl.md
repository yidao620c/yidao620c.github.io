---
title: 【安全贴士】证书操作工具OpenSSL
date: 2020-05-23 06:22:22 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 数字证书
  - 安全贴士
---

## 环境配置

1.修改openssl配置文件

通过vi编辑文件`/etc/pki/tls/openssl.cnf`，更新内容如下，有的部分是更新，有的是增加。

```ini
[ CA_default ]
req_extensions = v3_req

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ v3_server_client ]
basicConstraints        = critical, CA:FALSE
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always, issuer:always
keyUsage                = critical, nonRepudiation, digitalSignature, keyEncipherment, keyAgreement
extendedKeyUsage        = critical, serverAuth, clientAuth
crlDistributionPoints   = URI:https://example.com/xxx.crl

[ v3_sign ]
basicConstraints        = critical, CA:FALSE
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always, issuer:always
keyUsage                = critical, digitalSignature
extendedKeyUsage        = critical, codeSigning

[ v3_ca ]
basicConstraints = critical, CA:true
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
keyUsage = critical, cRLSign, digitalSignature, keyCertSign
crlDistributionPoints=URI:https://example.com/root.crl

[ crl_ext ]
authorityKeyIdentifier=keyid:always, issuer:always
```

**_强烈提醒：_**如果是要做客户端证书认证，证书类型一定要选`v3_server_client`

X.509 v3证书定义的扩展为用户或公钥以及在CA间的管理提供了额外的功能。X.509 v3证书的格式允许组织使用私有的扩展。 证书中的扩展被定义为`critical`或`non-critical`。
一个使用证书的系统在接收到设置为critical但无法识别或无法处理的证书时，必须拒绝处理该证书。 一个`non-critical`的证书在无法识别时可能会被忽略。

* 约束1：主机已经安装了OpenSSL，版本号1.0.2
* 约束2：生成CA根证书前，请提前规划以下环境配置参数。
<!--more-->

```bash
export root_key_password=123456;   #CA根私钥口令
export root_p12_password=123456;   #CA根证书库PKCS12口令
export server_key_password=123456; #服务证书私钥口令
export server_p12_password=123456; #服务证书库PKCS12口令
export sign_key_password=123456;   #签名证书私钥口令
```

2.初始化文件

```bash
CA_WORK_DIR=/etc/pki/CA
cd ${CA_WORK_DIR}
mkdir csr/ certs/ private/   #创建证书请求、证书、私钥文件夹
touch index.txt              #生成证书索引数据库文件
touch crlnumber              #吊销序列号文件
```

## 生成CA根证书

3.创建CA根私钥并指定加密密钥(PKCS8格式) 推荐做法是不要在命令行里面写密码，提示密码输入更佳。

```bash
(umask 077; openssl genrsa -aes256 -passout pass:${root_key_password} -out private/root.key 3072)
```

4.生成自签名X509 V3 CA根证书
`req`是证书请求的子命令，`-x509`表示输出证书，`-days365` 为有效期，此后根据提示输入证书拥有者信息。

```bash
openssl req -x509 -days 3650 -sha256 -extensions v3_ca -key private/root.key -passin pass:${root_key_password} -out root.crt \
-subj "/C=CN/ST=SX/L=XA/O=xncoding/OU=GTS/CN=ca.xncoding.com/emailAddress=ca@xncoding.com"
```

DN字段名                  | 缩写         | 说明                     | 填写要求
------------------------ |--------------|------------------------|-----------------------------------------
Country Name             | C            | 证书持有者所在国家         | 必填。要求填写国家代码，用2个字母表示
State or Province Name   | ST           | 证书持有者所在州或省份      | 可省略不填，填写全称
Locality Name            | L            | 证书持有者所在城市         | 可省略不填
Organization Name        | O            | 证书持有者所属组织或公司    | 可省略不填
Organizational Unit Name | OU           | 证书持有者所属部门         | 可省略不填
Common Name              | CN           | 证书持有者的通用名         | 必填。对于非应用证书，它应该在一定程度上具有惟一性；对于应用证书，一般填写服务器域名或通配符样式的域名。
Email Address            | emailAddress | 证书持有者的通信邮箱        | 可省略不填

5.查看证书内容

查看证书内容，下面的`-text参数`也可以是`-text|issuer|subject|serial|dates`
```bash
openssl x509 -in root.crt -noout -text
```

6.证书格式转换

```bash
#转换可以将一种类型的编码证书存入另一种。（即PEM到DER转换）
openssl x509 -in cert.crt -outform der-out cert.der
# DER到PEM
openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
```

7.仅导出根证书为PKCS12信任证书库（无私钥）

为了安全起见，给客户端使用的都应该是这种不带私钥的信任证书库。

```bash
openssl pkcs12 -export -nokeys -out root.p12 -in root.crt -certfile root.crt -passout pass:${root_p12_password} -certpbe AES-256-CBC
```

8.查看证书库的详情
```bash
openssl pkcs12 -info -in root.p12 -nodes -passin pass:${root_p12_password}
```

## 签发服务证书

9.生成Server私钥

生成RSA私钥(使用aes256加密)

```bash
(umask 077; openssl genrsa -aes256 -passout pass:${server_key_password} -out private/server.key 3072)
```

从私钥导出公钥文件（可选）

```bash
openssl rsa -in private/server.key -passin pass:${server_key_password} -pubout -out server.pub
```

10.生成Server证书请求

使用RSA私钥生成CSR签名请求

```bash
openssl req -new -extensions v3_server_client -key private/server.key -passin pass:${server_key_password} -out csr/server.csr \
-subj "/C=CN/ST=SX/L=XA/O=xncoding/OU=DC/CN=testserver/emailAddress=test@xncoding.com" \
-config /etc/pki/tls/openssl.cnf
```

11.生成服务的X509证书

```bash
openssl x509 -req -extensions v3_server_client -days 3650 -sha256 -CAcreateserial -CA root.crt \
-CAkey private/root.key -passin pass:${root_key_password} -in csr/server.csr -out certs/server.crt \
-extfile /etc/pki/tls/openssl.cnf
```

注`-CAcreateserial`会在本地创建一个`cacert.srl`文件，此后会依据这个文件中的序列号对证书进行序列号递增。

12.利用CA根证书校验证书

```bash
openssl verify -verbose -CAfile root.crt certs/server.crt
```

备注：如果有中间证书，则`root.crt`就是CA证书链，也就是需要把中间CA证书放到`root.crt`中，`server.crt`不需要证书链。

13.查询证书序列号

```bash
openssl x509 -in certs/server.crt -noout -serial | awk -F= '{print $2}'
```

查询出来的结果应该跟`root.srl`中的内容一样。

14.导出服务证书为PKCS12证书链

```bash
# 导出包含根证书的证书链
openssl pkcs12 -export -inkey private/server.key -passin pass:${server_key_password} \
-in certs/server.crt -chain -CAfile root.crt -out certs/server.p12 -password pass:${server_p12_password} 
```

对于有二级CA的场景，需要将二级CA加入到证书链中。

```bash
# 先生成CA证书链
cat middle.crt root.crt > chain.crt
openssl pkcs12 -export -inkey private/server.key -passin pass:${server_key_password} \
-in certs/server.crt -chain -CAfile chain.crt -out certs/server.p12 -password pass:${server_p12_password} 
```

15.直接拼接证书为证书链

```bash
cat certs/server.crt root.crt > certs/server_chain.crt
```

对于有二级CA的场景，需要将二级CA加入到证书链中。

```bash
cat certs/server.crt middle.crt root.crt > certs/server_chain.crt
```

## 证书吊销

16.获取到需要吊销的证书序列号

```bash
openssl x509 -in certs/server.crt -noout -serial -subject
```

结果显示

```
serial=85E94566A3335C975
subject= /C=CN/ST=SX/L=XA/O=xncoding/OU=DC/CN=testserver/emailAddress=test@xncoding.com
```

17.执行吊销命令

检查确认后，的确是要被吊销的证书，则执行吊销命令。

```bash
openssl ca -revoke certs/server.crt -keyfile private/root.key -passin pass:${root_key_password} \
-cert root.crt -config /etc/pki/tls/openssl.cnf
```

18.更新吊销列表

```bash
echo "85E94566A3335C975" > crlnumber #这个只需要在第一次更新证书吊销列表前，才需要执行
openssl ca -gencrl -keyfile private/root.key -passin pass:${root_key_password} \
-cert root.crt -out crl/crl.crt -config /etc/pki/tls/openssl.cnf
```

19.查看吊销列表：

```bash
# 如果CRL是DER格式的，则需要先转换成PEM格式
openssl crl -in xxx.der -inform der -outform pem -out xxx.crl
openssl crl -in crl/crl.crt -noout -text
openssl crl -in crl/crl.crt -noout -nextupdate
openssl crl -in crl/crl.crt -noout -text | grep "Serial Number:" | awk 'BEGIN{FS=": "} {print $2}'
# 或者一个管道命令搞定
openssl crl -in xxx.der -inform der -outform pem | openssl crl -noout -text
```

## 证书导入/导出

20.导出PKCS12证书

`-export`表示导出PKCS12证书，`-inkey` 指定了私钥文件，`-passin` 为私钥口令(nodes为无加密)，`-password`指定p12文件口令(导入导出)

```bash
openssl pkcs12 -export -inkey private/server.key -passin pass:${server_key_password} \
-in server.crt -password pass:${server_p12_password} -out certs/server.p12
```

导出待证书链的PKCS12格式证书

```bash
openssl pkcs12 -export -inkey private/server.key -passin pass:${server_key_password} \
-in certs/server.crt -chain -CAfile root.crt -out certs/server.p12 -password pass:${server_p12_password} 
```

21.PKCS12格式中的证书提取

```bash
#提取PEM文件(含私钥)
openssl pkcs12 -in server.p12 -password pass:${server_p12_password} -passout pass:${server_key_password} -out server.key
#仅提取私钥
openssl pkcs12 -in server.p12 -password pass:${server_p12_password} -passout pass:${server_key_password} -nocerts -out server.key
#检查私钥
openssl rsa -in server.key -check
#解密私钥key变成明文
openssl rsa -in server.key -out server.key
#仅提取证书(所有证书)
openssl pkcs12 -in server.p12 -password pass:${server_p12_password} -nokeys -out server.crt
#仅提取ca证书
openssl pkcs12 -in server-all.p12 -password pass:${server_p12_password} -nokeys -cacerts -out cacert.crt
#仅提取server证书
openssl pkcs12 -in server-all.p12 -password pass:${server_p12_password} -nokeys -clcerts -out server.crt
```

22.查看证书库中的内容

```bash
openssl pkcs12 -in certs/server.p12 -nodes -passin pass:${server_p12_password}
openssl pkcs12 -in certs/server.p12 -nodes -passin pass:${server_p12_password} | openssl x509 -noout -text
openssl pkcs12 -in certs/server.p12 -password pass:${server_p12_password} -nokeys | openssl x509 -noout -enddate
```

## 演示root.12校验server.p12

```bash
# 先提取出server.crt
openssl pkcs12 -nokeys -clcerts -in server.p12 -password pass:${server_p12_password} -out server.crt
# 然后提取出root.crt
openssl pkcs12 -nokeys -clcerts -in root.p12 -password pass:${root_p12_password} -out root.crt
# 使用root.crt校验server.crt
openssl verify -CAfile root.crt server.crt
```

## 默认证书的加密算法安全问题

默认的PKCS12和KeyStore算法都很旧了，需要升级到安全的算法。 参考<https://bugs.openjdk.java.net/browse/JDK-8228481>。 经测试，在OpenJDK
15版本上面修改配置后是可以的，也就是通过KeyTool工具来生成信任库即可。 预计等下个补丁版发布，所有的版本都默认支持了。

## 使用openssl查询服务端的证书链信息

```bash
openssl s_client -showcerts -connect HostIP:443
openssl s_client -showcerts -connect HostIP:443 </dev/null 2>/dev/null|openssl x509 -outform PEM >ca.crt
```

