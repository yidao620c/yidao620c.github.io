---
title: 【安全贴士】使用PSS填充的签名验签工具
date: 2020-05-29 09:12:09 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 加解密
  - 安全贴士
---

生成签名用的公私钥对

```bash
# 生成私钥
(umask 077; openssl genrsa -aes256 -out sign.key 3072)
# PKCS1私钥转换为PKCS8
openssl pkcs8 -topk8 -inform PEM -in sign.key -outform pem -out sign.key
# 从私钥导出公钥文件
openssl rsa -in sign.key -pubout -out sign.pub
```
<!--more-->

## 使用openssl命令行签名和验签

1）通过SHA256生成摘要

```bash
openssl dgst -sha256 file.txt
```

2）私钥签名，首先将你的文件通过摘要算法比如SHA256生成一个摘要，并对这个摘要进行私钥签名，输出签名文件。

```bash
# 对于OpenSSL算法库，签名时应该先调用RSA_padding_add_PKCS1_PSS进行PSS填充，
# 再对填充结果调用RSA_private_encrypt并指定RSA_NO_PADDING填充方式进行签名
# 验证签名时应该先调用RSA_public_decrypt进行公钥解密，再调用RSA_verify_PKCS1_PSS验证签名；
openssl dgst -sign sign.key -sigopt rsa_padding_mode:pss -sha256 -out sign.sha256 file.txt
openssl base64 -in sign.sha256 -out sign.txt
rm -f sign.sha256
```

3）公钥验证

```bash
openssl base64 -d -in sign.txt -out sign.sha256
openssl dgst -verify sign.pub -sha256 -sigopt rsa_padding_mode:pss -signature sign.sha256 file.txt
rm -f sign.txt sign.sha256
```

4）证书验证。首先将证书中的公钥提取出来，后面就跟公钥验证步骤一致了

```bash
openssl x509 -pubkey -noout -in sign.crt > sign.pub
```

## 使用Java编程进行签名和验签

### 引入BC依赖

```xml

<dependency>
    <groupId>org.bouncycastle</groupId>
    <artifactId>bcprov-jdk15on</artifactId>
    <version>1.62</version>
</dependency>
```

### 签名验签工具类

``` java
import java.io.File;
import java.io.FileInputStream;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.security.KeyFactory;
import java.security.MessageDigest;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.Security;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.security.spec.MGF1ParameterSpec;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.PSSParameterSpec;
import java.security.spec.X509EncodedKeySpec;
import java.util.Base64;
import javax.crypto.EncryptedPrivateKeyInfo;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.PBEKeySpec;

import org.bouncycastle.jce.provider.BouncyCastleProvider;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import sun.misc.BASE64Encoder;

/**
 * Description
 *
 * @author xiongneng
 * @since 2020-02-02
 */
public class RSASignature {

    private static final Logger Logger = LoggerFactory.getLogger(RSASignature.class);

    /**
     * 签名算法
     */
    private static final String SIGN_ALGORITHMS = "SHA256withRSA";
    private static final String SIGN_ALGORITHMS_PSS = "SHA256withRSA/PSS";
    private static final String ENCODE_ALGORITHM = "SHA-256";

    /**
     * 生成文件摘要
     *
     * @param content 文件内容
     * @param encode 字符集编码
     * @return 文件摘要
     */
    public static String digest(String content, String encode) {
        try {
            MessageDigest messageDigest = MessageDigest.getInstance(ENCODE_ALGORITHM);
            messageDigest.update(content.getBytes(encode));
            byte[] digestBytes = messageDigest.digest();
            return new String(Base64.getEncoder().encode(digestBytes), ICommon.GBK_ENCODEING);
        } catch (Exception e) {
            Logger.error("Summary calculation failed. {}", e.getMessage());
        }
        return null;
    }

    /**
     * RSA签名，结果写入签名值文件中
     *
     * @param file 待签名文件
     * @param signFile 签名值文件
     * @param signKeyFile 签名私钥文件
     * @param keyPassword 签名私钥口令
     */
    public static void sign(File file, String signFile, String signKeyFile, String keyPassword) {
        try {
            // 先解密私钥
            String encrypted = new String(Files.readAllBytes(Paths.get(signKeyFile)));
            encrypted = encrypted.replace("-----BEGIN ENCRYPTED PRIVATE KEY-----", "");
            encrypted = encrypted.replace("-----END ENCRYPTED PRIVATE KEY-----", "");
            encrypted = encrypted.replaceAll("\\n", "");
            EncryptedPrivateKeyInfo pkInfo = new EncryptedPrivateKeyInfo(Base64.getDecoder().decode(encrypted));
            PBEKeySpec keySpec = new PBEKeySpec(keyPassword.toCharArray()); // password
            SecretKeyFactory pbeKeyFactory = SecretKeyFactory.getInstance(pkInfo.getAlgName());
            PKCS8EncodedKeySpec encodedKeySpec = pkInfo.getKeySpec(pbeKeyFactory.generateSecret(keySpec));
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            PrivateKey priKey = keyFactory.generatePrivate(encodedKeySpec);

            // 开始签名
            Security.addProvider(new BouncyCastleProvider());
            java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS_PSS);
            signature.setParameter(new PSSParameterSpec(MGF1ParameterSpec.SHA256.getDigestAlgorithm(), "MGF1",
                    MGF1ParameterSpec.SHA256, 32, 1));
            signature.initSign(priKey);
            signature.update(Files.readAllBytes(file.toPath()));
            byte[] signed = signature.sign();
            String signStr = new String(Base64.getEncoder().encode(signed), StandardCharsets.UTF_8);
            System.out.println(signStr);
            Files.write(Paths.get(signFile), signStr.getBytes(StandardCharsets.UTF_8));
        } catch (Exception e) {
            Logger.error("digital signature failed", e);
        }
    }

    /**
     * RSA验签名检查
     *
     * @param originFile 待验证文件
     * @param signFile 签名值文件
     * @param signCertFile 签名证书文件
     * @return 是否验签成功
     */
    public static boolean verify(File originFile, String signFile, String signCertFile) {
        try {
            CertificateFactory fact = CertificateFactory.getInstance("X.509");
            X509Certificate certificate = (X509Certificate) fact.generateCertificate(new FileInputStream(signCertFile));
            PublicKey pk = certificate.getPublicKey();
            byte[] keyBytes = pk.getEncoded();
            String publicKey = new BASE64Encoder().encode(keyBytes);
            publicKey = publicKey.replaceAll("\\r?\\n", "");
            byte[] encodedKey = Base64.getDecoder().decode(publicKey);
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            PublicKey pubKey = keyFactory.generatePublic(new X509EncodedKeySpec(encodedKey));
            Security.addProvider(new BouncyCastleProvider());
            java.security.Signature signature = java.security.Signature.getInstance(SIGN_ALGORITHMS_PSS);
            signature.setParameter(new PSSParameterSpec(MGF1ParameterSpec.SHA256.getDigestAlgorithm(), "MGF1",
                    MGF1ParameterSpec.SHA256, 32, 1));
            signature.initVerify(pubKey);
            signature.update(Files.readAllBytes(originFile.toPath()));
            String signStr = new String(Files.readAllBytes(Paths.get(signFile)), StandardCharsets.UTF_8);
            return signature.verify(Base64.getDecoder().decode(signStr.replaceAll("\\n", "")));
        } catch (Exception e) {
            Logger.error("Digital signature verification failed. {}", e.getMessage());
        }
        return false;
    }
}
```

