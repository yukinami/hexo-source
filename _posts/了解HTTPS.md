title: 了解HTTPS
date: 2016-02-15 13:53:56
tags:
- Android
- Https
---

HTTPS是在基于TLS/SSL的安全套接字上的的应用层协议，除了传输层进行了加密外，其它与常规HTTP协议基本保持一致。

## SSL通信原理

当你的浏览器向服务器请求一个安全的网页(通常是 https://)
![](http://cdn.yeeyan.org/upload/attached/2011-02/23/20110223210236_77545.png)

<!--more-->

服务器就把它的证书和公匙发回来
![](http://cdn.yeeyan.org/upload/attached/2011-02/23/20110223210204_59544.png)

浏览器检查证书是不是由可以信赖的机构颁发的，确认证书有效和此证书是此网站的。
![](http://cdn.yeeyan.org/upload/attached/2011-02/23/20110223210220_99470.png)

使用公钥加密了一个随机对称密钥，包括加密的URL一起发送到服务器
![](http://cdn.yeeyan.org/upload/attached/2011-02/23/20110223210236_16109.png)

服务器用自己的私匙解密了你发送的钥匙。然后用这把对称加密的钥匙给你请求的URL链接解密。
![](http://cdn.yeeyan.org/upload/attached/2011-02/23/20110223210255_64661.png)

服务器用你发的对称钥匙给你请求的网页加密。你也有相同的钥匙就可以解密发回来的网页了
![](http://cdn.yeeyan.org/upload/attached/2011-02/23/20110223210218_70895.png)

因此为了实现SSL，我们的服务器需要安装私钥以及包含公钥的证书。其次我们需要让浏览器信任我们的证书，就是说有办法证明浏览器端实际获得的证书中包含的公钥确实是我们服务器的公钥。

第一点，私钥和证书可以通过openssl来生成。第二点，则需要通过证书签名来实现。

如果不能证明证书的有效性，那就可能会遭受中间人攻击（Man-in-the-middle attack，缩写：MITM）。

下面是攻击示例

假设爱丽丝（Alice）希望与鲍伯（Bob）通信。同时，马洛里（Mallory）希望拦截会话以进行窃听并可能在某些时候传送给鲍伯一个虚假的消息。

首先，爱丽丝会向鲍勃索取他的公钥。如果Bob将他的公钥发送给Alice，并且此时马洛里能够拦截到这个公钥，就可以实施中间人攻击。马洛里发送给爱丽丝一个伪造的消息，声称自己是鲍伯，并且附上了马洛里自己的公钥（而不是鲍伯的）。

爱丽丝收到公钥后相信这个公钥是鲍伯的，于是爱丽丝将她的消息用马洛里的公钥（爱丽丝以为是鲍伯的）加密，并将加密后的消息回给鲍伯。马洛里再次截获爱丽丝回给鲍伯的消息，并使用马洛里自己的私钥对消息进行解密，如果马洛里愿意，她也可以对消息进行修改，然后马洛里使用鲍伯原先发给爱丽丝的公钥对消息再次加密。当鲍伯收到新加密后的消息时，他会相信这是从爱丽丝那里发来的消息。

1. 爱丽丝发送给鲍伯一条消息，却被马洛里截获：

    爱丽丝“嗨，鲍勃，我是爱丽丝。给我你的公钥” --> 马洛里 鲍勃

2. 马洛里将这条截获的消息转送给鲍伯；此时鲍伯并无法分辨这条消息是否从真的爱丽丝那里发来的：

    爱丽丝 马洛里“嗨，鲍勃，我是爱丽丝。给我你的公钥” --> 鲍伯

3. 鲍伯回应爱丽丝的消息，并附上了他的公钥：

    爱丽丝 马洛里<-- [鲍伯的公钥]-- 鲍伯

4. 马洛里用自己的密钥替换了消息中鲍伯的密钥，并将消息转发给爱丽丝，声称这是鲍伯的公钥：

    爱丽丝<-- [马洛里的公钥]-- 马洛里 鲍勃

5. 爱丽丝用她以为是鲍伯的公钥加密了她的消息，以为只有鲍伯才能读到它：

    爱丽丝“我们在公共汽车站见面！”--[使用马洛里的公钥加密] --> 马洛里 鲍勃

6. 然而，由于这个消息实际上是用马洛里的密钥加密的，所以马洛里可以解密它，阅读它，并在愿意的时候修改它。他使用鲍伯的密钥重新加密，并将重新加密后的消息转发给鲍伯：

    爱丽丝 马洛里“在家等我！”--[使用鲍伯的公钥加密] --> 鲍伯

7. 鲍勃认为，这条消息是经由安全的传输通道从爱丽丝那里传来的。

## 证书签名

首先，我们需要CA对被验证方的原始证书进行签名（私钥加密），生成最终的证书。这里，CA是认证中心的英文 Certification Authority 的缩写。CA 认证中心，又称为数字证书认证中心或电子认证服务机构（比如 GlobalSign）。签名需要生成CSR，CSR是一个证书签名请求，在申请证书之前，首先要在WEB 服务器上生成 CSR 并将其提交给 CA 认证中心，CA 才能给您签发SSL数字证书 。可以这样认为，CSR 就是一个在您的服务器上生成的证书。CSR 包括以下内容：

a) 您的组织信息（组织名称、国家等等）
b) 您的Web服务器的域名


然后，验证方需要信任CA提供方自己的证书(CAcert)，比如证书在操作系统的受信任证书列表中，或者用户通过“安装根证书”等方式将 CA的公钥和私钥加入受信任列表。

这样，就当验证方接受到最终的证书后，通过查看它的签名CA信息，确定它的有效性，然后利用CAcert中包含的公钥进行解密，就得到被验证方的原始证书。

当然，证书可以不通过CA签证，而是自签名。但是这样，通常验证方就无法验证其有效性，只能无条件信任任何证书，或者将自签名的CA证书作为CAcert加入受信任列表。

### 证书链

证书颁发机构(CA)共分为两种类型：根CA和中间CA

![](https://support.dnsimple.com/files/dnsimple-ssl-chain-robowhois.png)

以下是是证书链的示例：

假设您从Awesome机构购买证书，域名是example.awesome。Awesome机构不是一个更证书颁发机构。换句话说，它的证书并不是直接嵌入在web浏览器，因此它不能被明确的信赖。

- Awesome机构使用证书由中间证书颁发机阿尔法颁发的证书
- 中间CA阿尔法使用由中间证书颁发机构贝塔颁发的证书
- 中间CA贝塔使用由中间证书颁发机伽马颁发的证书
- 中间Awesome CA伽马使用由The King of Awesomeness 颁发的证书
- The King of Awesomeness是一个根CA。该证书是直接嵌入在您的web浏览器中，因此可以被信任。

以上的例子中，SSL证书链是由以下6个证书组成的：
1. 终端证书：颁发给example.com, 发行商：Awesome机构
2. 中间证书1： 颁发给：example.com, 发行商： 中间颁发机构阿尔法
3. 中间证书2： 颁发给：中间证书颁发机构阿尔法, 发行商： 中间颁发机构阿尔法
4. 中间证书3： 颁发给：中间证书颁发机构贝塔, 发行商： 中间颁发机构伽马
5. 中间证书4：颁发给：中间证书颁发机构伽马： 发行商：The King of Awesomeness
6. 根证书： 颁发给：King of Awesomeness, 由The King of Awesomeness颁发

因此，通过证书链也可以证书证书的有效性。

如果证书是由中间CA签名的，那么服务器端需要安装证书链。

配置服务端证书链时，有两点需要注意：
1. 证书是在握手期间发送的，由于 TCP 初始拥塞窗口的存在，如果证书太长可能会产生额外的往返开销
2. 如果证书没包含中间证书，大部分浏览器可以正常工作，但会暂停验证并根据子证书指定的父证书URL自己获取中间证书。这个过程会产生额外的 DNS 解析、建立 TCP 连接等开销，非常影响性能。 这里父证书的URL是从证书的[AIA扩展信息][authority-information-access]（Authority Information Access）中获取的。

## 实践

### 证书的生成

证书的生成可以用linux的OpenSSL工具链。对于一个网站，首先必须有自己的私钥，私钥的生成方式为：

```
openssl genrsa -out ssl.key 2048
```

### 生成证书请求

```
openssl req -new -key ssl.key -out ssl.csr
```

### 自签名，生成CA证书

CA证书是一种特殊的自签名证书，可以用来对其它证书进行签名。这样当验证方选择信任了CA证书，被签名的其它证书就被信任了。在验证方进行验证时，CA证书来自操作系统的信任证书库，或者指定的证书列表。

```
openssl x509 -req -in ssl.csr -extensions v3_ca -signkey sign.key -out sign.crt
```

### 利用CA证书进行签名

```
openssl x509 -req -in ssl.csr -extensions v3_usr -CA sign.crt -CAkey sign.key -CAcreateserial -out ssl.crt
````

当然，也可以不生成自签名CA证书进行签名，直接生成自签名证书

```
openssl x509 -req -in ssl.csr -signkey ssl.key -out ssl.crt
```

[tls-and-certificates]: http://www.cnblogs.com/kyrios/p/tls-and-certificates.html
[what-is-ssl-certificate-chain]: https://support.dnsimple.com/articles/what-is-ssl-certificate-chain/
[authority-information-access]: https://www.tbs-certificates.co.uk/FAQ/en/453.html
[optimize-tls-handshake]: https://imququ.com/post/optimize-tls-handshake.html