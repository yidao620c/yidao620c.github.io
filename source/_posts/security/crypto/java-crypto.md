---
title: 【安全贴士】Java加解密算法
date: 2020-05-25 22:12:09 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 加解密
  - 安全贴士
---

密码学领域有对称加密和非对称加密算法，本篇将使用Java语言来实现几种常见的加解密算法。

完整源码请参考 [GitHub源码](https://github.com/yidao620c/thinking-java/tree/master/src/main/java/ch02utils/crypto)

对称加密算法概念

加密密钥和解密密钥相同，大部分算法加密揭秘过程互逆。

特点：算法公开、（相比非对称加密）计算量小、加密速度快、效率高。

弱点：双方都使用同样的密钥，安全性得不到保证。

常用对称加密算法

* DES（Data Encryption Standard）
* 3DES（DES加强版，使用3次DES计算，Triple DES，DESede）
* AES（Advanced Encryption Standard，3DES加强版）

这里只列出最常用的AES算法实现，也是安全等级最高，推荐使用的对称加密算法，其他实现请参考我的GitHub上的源码。
<!--more-->

``` java
/**
 * 对称加密/解密算法（推荐算法）：AES
 *
 * @author XiongNeng
 * @version 1.0
 * @since 2017-11-22
 */
public class AESUtil {
    private static final String DATA = "这个是内容";
    private static final String KEY_ALGORITHM = "AES";
    private static final String CIPHER_ALGORITHM = "AES/ECB/PKCS5Padding";
    private static final Charset UTF8 = Charset.forName("UTF-8");

    public static byte[] encrypt(String data, String key) throws Exception {
        return encrypt(data.getBytes(UTF8), key.getBytes(UTF8));
    }

    public static byte[] encrypt(String data, byte[] key) throws Exception {
        return encrypt(data.getBytes(UTF8), key);
    }

    public static byte[] encrypt(byte[] data, String key) throws Exception {
        return encrypt(data, key.getBytes(UTF8));
    }

    public static byte[] encrypt(byte[] data, byte[] key) throws Exception {
        Key k = genSecretKey(key);
        Cipher cipher = Cipher.getInstance(CIPHER_ALGORITHM);
        cipher.init(Cipher.ENCRYPT_MODE, k);
        return cipher.doFinal(data);
    }

    public static byte[] decrypt(String data, String key) throws Exception {
        return decrypt(data.getBytes(UTF8), key.getBytes(UTF8));
    }

    public static byte[] decrypt(String data, byte[] key) throws Exception {
        return decrypt(data.getBytes(UTF8), key);
    }

    public static byte[] decrypt(byte[] data, String key) throws Exception {
        return decrypt(data, key.getBytes(UTF8));
    }

    public static byte[] decrypt(byte[] data, byte[] key) throws Exception {
        Key k = genSecretKey(key);
        Cipher cipher = Cipher.getInstance(CIPHER_ALGORITHM);
        cipher.init(Cipher.DECRYPT_MODE, k);
        return cipher.doFinal(data);
    }

    /**
     * AES only supports key sizes of 16, 24 or 32 bytes
     */
    public static Key genSecretKey(byte[] key) throws Exception {
        if (key.length == 16 || key.length == 24 || key.length == 32) {
            return new SecretKeySpec(key, KEY_ALGORITHM);
        }
        throw new IllegalArgumentException("AES only supports key sizes of 16, 24 or 32 bytes");
    }

    public static void main(String[] args) throws Exception {
        String key = "1234567890123456";
        byte[] desResult = encrypt(DATA, key);
        System.out.println(DATA + ">>>AES 加密结果>>>" + Hex.encodeHexString(desResult));

        byte[] desPlain = decrypt(desResult, key);
        System.out.println(DATA + ">>>AES 解密结果>>>" + new String(desPlain, Charset.forName("UTF-8")));
    }
}
```

## 非对称加密算法

**非对称密码概念**

发送者使用接收者的公钥加密，接收者使用自己的私钥解密。需要两个密钥进行加密或解密，分为公钥和私钥。

特点：安全性高，速度慢

常用算法

* RSA算法
* DH密钥交换算法
* ElGamal算法那

用途：

* 密钥交换
* 加密/解密
* 数字签名

RSA算法示例：

``` java
public class RSAUtil {
    // 非对称密钥算法
    private static final String KEY_ALGORITHM = "RSA";
    // 密钥长度必须是64的倍数，在512到65536位之间
    private static final int KEY_SIZE = 2048;
    // 公钥
    private static final String PUBLIC_KEY = "RSAPublicKey";
    // 私钥
    private static final String PRIVATE_KEY = "RSAPrivateKey";
    // 字符编码
    private static final Charset UTF8 = Charset.forName("UTF-8");

    /**
     * 私钥加密
     *
     * @param dataStr 待加密数据
     * @param key  密钥
     * @return byte[] 加密数据
     */
    public static byte[] encryptByPrivateKey(String dataStr, byte[] key) throws Exception {
        //取得私钥
        PKCS8EncodedKeySpec pkcs8KeySpec = new PKCS8EncodedKeySpec(key);
        KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
        //生成私钥
        PrivateKey privateKey = keyFactory.generatePrivate(pkcs8KeySpec);
        //数据加密
        Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
        cipher.init(Cipher.ENCRYPT_MODE, privateKey);
        return cipher.doFinal(dataStr.getBytes(UTF8));
    }

    /**
     * 公钥加密
     *
     * @param dataStr 待加密数据
     * @param key  密钥
     * @return byte[] 加密数据
     */
    public static byte[] encryptByPublicKey(String dataStr, byte[] key) throws Exception {

        //实例化密钥工厂
        KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
        //初始化公钥，密钥材料转换
        X509EncodedKeySpec x509KeySpec = new X509EncodedKeySpec(key);
        //产生公钥
        PublicKey pubKey = keyFactory.generatePublic(x509KeySpec);

        //数据加密
        Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
        cipher.init(Cipher.ENCRYPT_MODE, pubKey);
        return cipher.doFinal(dataStr.getBytes(UTF8));
    }

    /**
     * 私钥解密
     *
     * @param data 待解密数据
     * @param key  密钥
     * @return String 解密数据
     */
    public static String decryptByPrivateKey(byte[] data, byte[] key) throws Exception {
        //取得私钥
        PKCS8EncodedKeySpec pkcs8KeySpec = new PKCS8EncodedKeySpec(key);
        KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
        //生成私钥
        PrivateKey privateKey = keyFactory.generatePrivate(pkcs8KeySpec);
        //数据解密
        Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        return new String(cipher.doFinal(data), UTF8);
    }

    /**
     * 公钥解密
     *
     * @param data 待解密数据
     * @param key  密钥
     * @return String 解密数据
     */
    public static String decryptByPublicKey(byte[] data, byte[] key) throws Exception {
        //实例化密钥工厂
        KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
        //初始化公钥，密钥材料转换
        X509EncodedKeySpec x509KeySpec = new X509EncodedKeySpec(key);
        //产生公钥
        PublicKey pubKey = keyFactory.generatePublic(x509KeySpec);
        //数据解密
        Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
        cipher.init(Cipher.DECRYPT_MODE, pubKey);
        return new String(cipher.doFinal(data), UTF8);
    }

    /**
     * 初始化密钥对
     *
     * @return Map 甲方密钥的Map
     */
    private static Map<String, Object> initKey() throws Exception {
        //实例化密钥生成器
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance(KEY_ALGORITHM);
        //初始化密钥生成器
        keyPairGenerator.initialize(KEY_SIZE);
        //生成密钥对
        KeyPair keyPair = keyPairGenerator.generateKeyPair();
        //甲方公钥
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        //甲方私钥
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        //将密钥存储在map中
        Map<String, Object> keyMap = new HashMap<String, Object>();
        keyMap.put(PUBLIC_KEY, publicKey);
        keyMap.put(PRIVATE_KEY, privateKey);
        return keyMap;
    }

    /**
     * 取得私钥
     *
     * @param keyMap 密钥map
     * @return byte[] 私钥
     */
    private static byte[] getPrivateKey(Map<String, Object> keyMap) {
        Key key = (Key) keyMap.get(PRIVATE_KEY);
        return key.getEncoded();
    }

    /**
     * 取得公钥
     *
     * @param keyMap 密钥map
     * @return byte[] 公钥
     */
    private static byte[] getPublicKey(Map<String, Object> keyMap) throws Exception {
        Key key = (Key) keyMap.get(PUBLIC_KEY);
        return key.getEncoded();
    }

    /**
     * @param args args
     * @throws Exception ex
     */
    public static void main(String[] args) throws Exception {
        //初始化密钥，生成密钥对
        Map<String, Object> keyMap = initKey();
        //公钥
        byte[] publicKey = getPublicKey(keyMap);

        //私钥
        byte[] privateKey = getPrivateKey(keyMap);
        System.out.println("公钥：" + Base64.encodeBase64String(publicKey));
        System.out.println("私钥：" + Base64.encodeBase64String(privateKey));

        System.out.println("================密钥对构造完毕,甲方将公钥公布给乙方，开始进行加密数据的传输=============");
        String str = "RSA密码交换算法";
        System.out.println("===========甲方向乙方发送加密数据==============");
        System.out.println("原文:" + str);
        //甲方进行数据的加密
        byte[] code1 = encryptByPrivateKey(str, privateKey);
        System.out.println("加密后的数据：" + Base64.encodeBase64String(code1));
        System.out.println("===========乙方使用甲方提供的公钥对数据进行解密==============");
        //乙方进行数据的解密
        String decode1 = decryptByPublicKey(code1, publicKey);
        System.out.println("乙方解密后的数据：" + decode1);

        System.out.println("===========反向进行操作，乙方向甲方发送数据==============");

        str = "乙方向甲方发送数据RSA算法";

        System.out.println("原文:" + str);

        //乙方使用公钥对数据进行加密
        byte[] code2 = encryptByPublicKey(str, publicKey);
        System.out.println("===========乙方使用公钥对数据进行加密==============");
        System.out.println("加密后的数据：" + Base64.encodeBase64String(code2));

        System.out.println("=============乙方将数据传送给甲方======================");
        System.out.println("===========甲方使用私钥对数据进行解密==============");

        //甲方使用私钥对数据进行解密
        String decode2 = decryptByPrivateKey(code2, privateKey);

        System.out.println("甲方解密后的数据：" + decode2);
    }
}
```

## 算法选择

RSA 算法规定：

1. 待加密的字节数不能超过密钥的长度值除以 8 再减去 11（即：KeySize / 8 - 11），
1. 加密后得到密文的字节数，正好是密钥的长度值除以 8（即：KeySize / 8）

也就是对于常见的2048位的密钥，最大加密长度为256 - 11 = 245个字节

针对超长问题，常见的解决办法有两种，

一、对待加密数据进行拆分，分成众多小片段，解密时，也分段解密。

二、结合对称加密算法使用，使用RSA加密对称加密算法的密钥。

RSA是企业级应用标准，很多第三方的加密软件使用RSA 2048bit加密

优点：密码分配简单，安全保障性高

缺点：

1. 速度慢，RSA最快的情况也比DES慢上好几倍，RSA的速度比对应同样安全级别的对称密码算法要慢1000倍左右
2. 一般来说只用于少量数据加密
3. 产生密钥很麻烦，受到素数产生技术的限制，因而难以做到一次一密。

实际上，这些缺点是非对称加密本身的局限。

对称加密和非对称加密算法各自用途和功能以及非常明确了。实际内容传输都会使用对称加密， 而安全的交换对称密钥、数字签名就使用非对称加密。

