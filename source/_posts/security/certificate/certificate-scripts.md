---
title: 【安全贴士】一键生成证书脚本
date: 2020-05-17 09:10:19 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 数字证书
  - 安全贴士
---

这里演示如何通过命令行一键生成自签名CA证书，以及对于已有的证书生成服务证书库和信任证书库。
<!--more-->

```bash
#!/usr/bin/env bash
####################################################
# 一键生成证书库server.p12和信任证书库root.p12
# 证书库由openssl命令生成，信任证书库由keytool工具生成。
# ./x509.sh
####################################################

# 定义变量
export CA_WORK_DIR=`pwd`
export config="${CA_WORK_DIR}/myopenssl.cnf"
export root_key_password=123456
export root_p12_password=123456
export middle_key_password=123456
export server_key_password=123456
export server_p12_password=123456
export sign_key_password=123456
export cn="test20201229"
export days=3550

# 生成必要的文件
[[ -d private ]] || mkdir private
[[ -d csr ]] || mkdir csr
[[ -d certs ]] || mkdir certs 
# 初始化索引文件
[[ -f index.txt ]] || touch index.txt
# 吊销序列号文件
[[ -f index.txt ]] || touch crlnumber

# 初始化openssl配置
grep "v3_server_client" $config &>/dev/null
if [[ "$?" != "0" ]]; then
    cp /etc/pki/tls/openssl.cnf $config
    cat <<EOF >>$config

[ v3_rootca ]
basicConstraints = critical, CA:true
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
keyUsage = critical, cRLSign, digitalSignature, keyCertSign
crlDistributionPoints=URI:https://example.com/rootca.crl

[ v3_middleca ]
basicConstraints = critical, CA:true
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
keyUsage = critical, cRLSign, digitalSignature, keyCertSign
crlDistributionPoints=URI:https://example.com/middleca.crl

[ v3_server_client ]
basicConstraints        = critical, CA:FALSE
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always, issuer:always
keyUsage                = critical, nonRepudiation, digitalSignature, keyEncipherment, keyAgreement
extendedKeyUsage        = critical, serverAuth, clientAuth
crlDistributionPoints   = URI:https://example.com/server.crl

[ v3_sign ]
basicConstraints        = critical, CA:FALSE
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always, issuer:always
keyUsage                = critical, digitalSignature
extendedKeyUsage        = critical, codeSigning
EOF
fi

# 生成CA根证书
if [[ ! -f private/root.key ]]; then
    (umask 077; openssl genrsa -aes256 -passout pass:${root_key_password} -out private/root.key 3072)
    openssl req -x509 -days 3650 -sha256 -extensions v3_rootca -key private/root.key -passin pass:${root_key_password} \
        -out root.crt -subj "/C=CN/ST=SX/L=XA/O=xncoding/OU=GTS/CN=root.xncoding.com/emailAddress=root@xncoding.com" \
        -config ${config}
    keytool -import -noprompt -trustcacerts -alias rootca -file root.crt -keystore root.p12 \
        -storetype PKCS12 -storepass ${root_p12_password}
fi

# 生成二级CA证书
if [[ ! -f private/middle.key ]]; then
    (umask 077; openssl genrsa -aes256 -passout pass:${middle_key_password} -out private/middle.key 3072)
    openssl req -new -extensions v3_middleca -key private/middle.key -passin pass:${middle_key_password} \
        -out csr/middle.csr -subj "/C=CN/ST=SX/L=XA/O=xncoding/OU=DC/CN=middle/emailAddress=middle@xncoding.com" \
        -config ${config}
    openssl x509 -req -extensions v3_middleca -days ${days} -sha256 -CAcreateserial -CA root.crt \
        -CAkey private/root.key -passin pass:${root_key_password} -in csr/middle.csr -out middle.crt \
        -extfile ${config}
fi

# 签发服务证书
(umask 077; openssl genrsa -aes256 -passout pass:${server_key_password} -out private/server.key 3072)
openssl req -new -extensions v3_server_client -key private/server.key -passin pass:${server_key_password} \
    -out csr/server.csr -subj "/C=CN/ST=SX/L=XA/O=xncoding/OU=DC/CN=${cn}/emailAddress=${cn}@xncoding.com" \
    -config ${config}
openssl x509 -req -extensions v3_server_client -days ${days} -sha256 -CAcreateserial -CA middle.crt \
    -CAkey private/middle.key -passin pass:${middle_key_password} -in csr/server.csr -out certs/server.crt \
    -extfile ${config}

# 导出服务证书为PKCS12的证书链
[[ -f cachain.crt ]] || cat middle.crt root.crt > cachain.crt
openssl pkcs12 -export -inkey private/server.key -passin pass:${server_key_password} \
    -in certs/server.crt -chain -CAfile cachain.crt -out certs/server.p12 -password pass:${server_p12_password} \
    -keypbe aes-256-cbc -certpbe aes-256-cbc

# 最后删除临时文件
rm -f csr/server.csr csr/middle.csr

# 查看证书库信息
# openssl pkcs12 -info -in certs/server.p12 -nodes -passin pass:${server_p12_password}
```