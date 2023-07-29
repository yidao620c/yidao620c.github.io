---
title: 【安全贴士】TrustManager加载默认的JRE信任证书
date: 2020-05-27 08:38:56 +0800
toc: true
categories: [ 网络安全 ]
tags:
  - 数字证书
  - 安全贴士
---

Essentially, get hold of the default trust manager, create a second trust manager that uses your own trust store. Wrap
them both in a custom trust manager implementation that delegates call to both (falling back on the other when one
fails).

参考文章 <https://stackoverflow.com/questions/24555890/using-a-custom-truststore-in-java-as-well-as-the-default-one>

<!--more-->

``` java
TrustManagerFactory tmf = TrustManagerFactory
    .getInstance(TrustManagerFactory.getDefaultAlgorithm());
// Using null here initialises the TMF with the default trust store.
tmf.init((KeyStore) null);

// Get hold of the default trust manager
X509TrustManager defaultTm = null;
for (TrustManager tm : tmf.getTrustManagers()) {
    if (tm instanceof X509TrustManager) {
        defaultTm = (X509TrustManager) tm;
        break;
    }
}

FileInputStream myKeys = new FileInputStream("truststore.jks");

// Do the same with your trust store this time
// Adapt how you load the keystore to your needs
KeyStore myTrustStore = KeyStore.getInstance(KeyStore.getDefaultType());
myTrustStore.load(myKeys, "password".toCharArray());

myKeys.close();

tmf = TrustManagerFactory
    .getInstance(TrustManagerFactory.getDefaultAlgorithm());
tmf.init(myTrustStore);

// Get hold of the default trust manager
X509TrustManager myTm = null;
for (TrustManager tm : tmf.getTrustManagers()) {
    if (tm instanceof X509TrustManager) {
        myTm = (X509TrustManager) tm;
        break;
    }
}

// Wrap it in your own class.
final X509TrustManager finalDefaultTm = defaultTm;
final X509TrustManager finalMyTm = myTm;
X509TrustManager customTm = new X509TrustManager() {
    @Override
    public X509Certificate[] getAcceptedIssuers() {
        // If you're planning to use client-cert auth,
        // merge results from "defaultTm" and "myTm".
        return finalDefaultTm.getAcceptedIssuers();
    }

    @Override
    public void checkServerTrusted(X509Certificate[] chain,
            String authType) throws CertificateException {
        try {
            finalMyTm.checkServerTrusted(chain, authType);
        } catch (CertificateException e) {
            // This will throw another CertificateException if this fails too.
            finalDefaultTm.checkServerTrusted(chain, authType);
        }
    }

    @Override
    public void checkClientTrusted(X509Certificate[] chain,
            String authType) throws CertificateException {
        // If you're planning to use client-cert auth,
        // do the same as checking the server.
        finalDefaultTm.checkClientTrusted(chain, authType);
    }
};


SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(null, new TrustManager[] { customTm }, null);

// You don't have to set this as the default context,
// it depends on the library you're using.
SSLContext.setDefault(sslContext);
```

You don't have to set that context as the default context. How you use it depends on the client library you're using (
and where it gets its socket factories from).

This being said, in principle, you'd always have to update the truststore as required anyway. The Java 7 JSSE Reference
Guide had an "important note" about this, now downgraded to just a "note" in version 8 of the same guide:

The JDK ships with a limited number of trusted root certificates in the
`java-home/lib/security/cacerts` file. As documented in keytool reference pages, it is your responsibility to maintain (
that is, add and remove) the certificates contained in this file if you use this file as a truststore.

Depending on the certificate configuration of the servers that you contact, you may need to add additional root
certificates. Obtain the needed specific root certificates from the appropriate vendor.