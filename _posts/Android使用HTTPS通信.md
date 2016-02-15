title: Android使用HTTPS通信
date: 2016-02-15 15:34:25
tags:
- Android
- Https
---

Android使用HTTPS通信首先需要告诉底层的HTTP客户端，对应https的Schema，使用什么端口，进行SSLSocket的通信。

这里以Httpclient为例

```
SSLSocketFactory  sf = new SSLSocketFactory();
sf.setHostnameVerifier(SSLSocketFactory.BROWSER_COMPATIBLE_HOSTNAME_VERIFIER);

HttpParams params = new BasicHttpParams();
HttpProtocolParams.setVersion(params, HttpVersion.HTTP_1_1);
HttpProtocolParams.setContentCharset(params, HTTP.UTF_8);

SchemeRegistry registry = new SchemeRegistry();
registry.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), 80));
registry.register(new Scheme("https", sf, 443));

ClientConnectionManager ccm = new ThreadSafeClientConnManager(params, registry);

return new DefaultHttpClient(ccm, params);
```

<!--more-->

HostnameVerifier有下面几个策略

- ALLOW_ALL_HOSTNAME_VERIFIER 关闭host验证，允许和所有的host简历SSL通信
- BROWSER_COMPATIBLE_HOSTNAME_VERIFIER  和浏览器兼容的验证策略，即通配符能够匹配所有子域名
- StrictHostnameVerifier 严格匹配模式，hostname必须匹配第一个CN或者任何一个subject-alts

为了避免中间人攻击，首先是需要打开host验证，其次打开了host验证后，那么客户端必须能够验证证书的有效性，否则在请求时会报如下错误

```
javax.net.ssl.SSLHandshakeException: java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.
        at org.apache.harmony.xnet.provider.jsse.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:374)
        at libcore.net.http.HttpConnection.setupSecureSocket(HttpConnection.java:209)
        at libcore.net.http.HttpsURLConnectionImpl$HttpsEngine.makeSslConnection(HttpsURLConnectionImpl.java:478)
        at libcore.net.http.HttpsURLConnectionImpl$HttpsEngine.connect(HttpsURLConnectionImpl.java:433)
        at libcore.net.http.HttpEngine.sendSocketRequest(HttpEngine.java:290)
        at libcore.net.http.HttpEngine.sendRequest(HttpEngine.java:240)
        at libcore.net.http.HttpURLConnectionImpl.getResponse(HttpURLConnectionImpl.java:282)
        at libcore.net.http.HttpURLConnectionImpl.getInputStream(HttpURLConnectionImpl.java:177)
        at libcore.net.http.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:271)
```

出现上述错误，通常是下面几个原因：

1. 签名证书的CA是个未知的CA
2. 服务器证书不是由CA签名的，而是自签名的
3. 服务器配置缺少中间CA

## 未知的CA

出现这种情况的原因是签名CA没有被系统所信任。这种情况可以告诉`HttpsURLConnection`来信任CA。

下面的例子从`InputStream`读取CA证书，用来创建`KeyStore`，然后用它来创建和初始化`TrustManager`。`TrustManager`是系统用来验证从服务器获取证书的。

```
// Load CAs from an InputStream
// (could be from a resource or ByteArrayInputStream or ...)
CertificateFactory cf = CertificateFactory.getInstance("X.509");
// From https://www.washington.edu/itconnect/security/ca/load-der.crt
InputStream caInput = new BufferedInputStream(new FileInputStream("load-der.crt"));
Certificate ca;
try {
    ca = cf.generateCertificate(caInput);
    System.out.println("ca=" + ((X509Certificate) ca).getSubjectDN());
} finally {
    caInput.close();
}

// Create a KeyStore containing our trusted CAs
String keyStoreType = KeyStore.getDefaultType();
KeyStore keyStore = KeyStore.getInstance(keyStoreType);
keyStore.load(null, null);
keyStore.setCertificateEntry("ca", ca);

// Create a TrustManager that trusts the CAs in our KeyStore
String tmfAlgorithm = TrustManagerFactory.getDefaultAlgorithm();
TrustManagerFactory tmf = TrustManagerFactory.getInstance(tmfAlgorithm);
tmf.init(keyStore);

// Create an SSLContext that uses our TrustManager
SSLContext context = SSLContext.getInstance("TLS");
context.init(null, tmf.getTrustManagers(), null);

// Tell the URLConnection to use a SocketFactory from our SSLContext
URL url = new URL("https://certs.cac.washington.edu/CAtest/");
HttpsURLConnection urlConnection =
    (HttpsURLConnection)url.openConnection();
urlConnection.setSSLSocketFactory(context.getSocketFactory());
InputStream in = urlConnection.getInputStream();
copyInputStreamToOutputStream(in, System.out);
```

NOTE: 这里如果传入的`TrustManager`什么都没有做，那么相当于还是没有加密通信，因为它会接受任何的证书，仍可能遭受中间人攻击。

`KeyStore`也可以用JDK的keytool工具生成

```
keytool -import -file ssl.crt  -storetype BKS -provider org.bouncycastle.jce.provider.BouncyCastleProvider -providerpath ./bouncycastle.jar -storetype BKS -keystore ssl.truststore
```

需要注意的是，Android默认使用BKS作为KeyStore Type。


下面的Httpclient的例子

```
InputStream keyStoreStream = this.getClass().getClassLoader().getResourceAsStream("ssl.truststore");

KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());
char[] key = "XXX".toCharArray();
trustStore.load(keyStoreStream,key);

SSLSocketFactory  sf = new SSLSocketFactory(trustStore);
sf.setHostnameVerifier(SSLSocketFactory.BROWSER_COMPATIBLE_HOSTNAME_VERIFIER);

HttpParams params = new BasicHttpParams();
HttpProtocolParams.setVersion(params, HttpVersion.HTTP_1_1);
HttpProtocolParams.setContentCharset(params, HTTP.UTF_8);

SchemeRegistry registry = new SchemeRegistry();
registry.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), 80));
registry.register(new Scheme("https", sf, 443));

ClientConnectionManager ccm = new ThreadSafeClientConnManager(params, registry);

return new DefaultHttpClient(ccm, params);
```

## 自签名证书

自签名证书以为着服务器担当自己的CA，这个未知的CA类似的，可以把自签名证书作为信任的CA导入TrustManager。

## 缺少中间CA

- 服务器把证书链安装完整
- 把未知的中间CA作为信任的未知CA导入TrustManager


[SSLSocketFactory]: https://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/conn/ssl/SSLSocketFactory.html
[security-ssl]: http://developer.android.com/training/articles/security-ssl.html