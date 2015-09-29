title: 用spring social构建API的client端
date: 2015-01-06 21:26:02
tags:
 - Spring Social
---

介绍
====
Spring Social可以用来连接Software-as-a-Service (SaaS) 服务商提供了REST API，Spring的项目本身已经提供了对Facebook, Twitter, Linkedin, Github等服务商的REST API的连接。
实际在开发app的过程中，通常是需要本地app连接到后台服务器的REST API的。客户端的开发本身可能还会涉及相对复杂的验证机制，比如`OAuth Dance`（客户端和服务器来回交互获取access token的过程），Spring Social为客户端的开发提供了一套框架，包括连接的建立，对API的强类型绑定，应用可以直接通过客户端访问需要的API。

<!--more-->

术语
=====
`spring-social-core`模块提供了一套***Service Provider Connect Framework***框架来实现到Software-as-a-Service (SaaS) 提供商的连接。

核心API
-------
###Connection
`Connection`接口定义了连接到外部`service provider`的模型
```
public interface Connection<A> extends Serializable {
    ConnectionKey getKey();
    String getDisplayName();
    String getProfileUrl();
    String getImageUrl();
    void sync();
    boolean test();
    boolean hasExpired();
    void refresh();
    UserProfile fetchUserProfile();
    void updateStatus(String message);
    A getApi();
    ConnectionData createData();
}
```
通过这个`Connection`可以获取到关于这个连接的一些元数据信息。最主要的是可以通过`Connection#getApi()`获取到远程API的本地Java binding实例。

建立connections
---------------
本地用户和service provider的连接的建立是通过`ServiceProvider`来完成的。每个认证协议都作为它的一个具体的实现，这把协议相关的内容隔离出了核心API。`ConnectionFactory`封装了使用特定的认证协议对`Connection`的构建。

###OAuth2 service providers
`OAuth2ConnectionFactory`用来建立基于OAuth2服务的连接
```
public class OAuth2ConnectionFactory<A> extends ConnectionFactory<A> {
    public OAuth2Operations getOAuthOperations();
    public Connection<A> createConnection(AccessGrant accessGrant);
    public Connection<A> createConnection(ConnectionData data);
    public void setScope(String scope);
    public String getScope();
    public String generateState();
    public boolean supportsStateParameter();
}
```
`getOAuthOperations()`返回一个API用来执行验证流程`OAuth Dance`。这个流程的最终结果是需要得到一个`AccessGrant`，从而能够通过调用`createConnection()`建立`connection`.

`OAuth2Operations`接口定义如下：
```
public interface OAuth2Operations {
    String buildAuthorizeUrl(OAuth2Parameters parameters);
    String buildAuthorizeUrl(GrantType grantType, OAuth2Parameters parameters);
    String buildAuthenticateUrl(OAuth2Parameters parameters);
    String buildAuthenticateUrl(GrantType grantType, OAuth2Parameters parameters);
    AccessGrant exchangeForAccess(String authorizationCode, String redirectUri,
        MultiValueMap<String, String> additionalParameters);
    AccessGrant exchangeCredentialsForAccess(String username, String password,
        MultiValueMap<String, String> additionalParameters);
    AccessGrant refreshAccess(String refreshToken,
        MultiValueMap<String, String> additionalParameters);
    AccessGrant authenticateClient();
    AccessGrant authenticateClient(String scope);
}
```

OAuth2的认证流程请[参考][1]，实际认证的流程类似：
```
FacebookConnectionFactory connectionFactory =
    new FacebookConnectionFactory("clientId", "clientSecret");
OAuth2Operations oauthOperations = connectionFactory.getOAuthOperations();
OAuth2Parameters params = new OAuth2Parameters();
params.setRedirectUri("https://my-callback-url");
String authorizeUrl = oauthOperations.buildAuthorizeUrl(params);
response.sendRedirect(authorizeUrl);

// upon receiving the callback from the provider:
AccessGrant accessGrant = oauthOperations.exchangeForAccess(authorizationCode, "https://my-callback-url", null);
Connection<Facebook> connection = connectionFactory.createConnection(accessGrant);
```



开发
====
项目结构
-------
Spring Social建议客户端模块以`org.springframework.social.{providerId}`为基础命名空间，比如说facebook的命名空间`org.springframework.social.facebook`。
在命名空间内部，推荐以下的子包目录结构：

|  子包 | 描述 |
|:-----------:|:--------------------------------------:|
|  api        |  定了API绑定的公共接口   |
|  api.impl   |  API绑定接口的实现       |
|  connect    |  建立到service providerd的连接所需要的类  |

开发provider’s API的Java binding
-------------------------------
###设计Java API binding
对于API绑定的涉及和实现，开发者是有完全的控制权的，但是框架对提高代码的一致性和质量方面提供了一些指导方针。
- 支持API绑定接口和实现分离
- 支持RESTful资源风格的方式组织分层的API binding

下面是Twitter API Binding的例子：
```
public interface Twitter extends ApiBinding {
    BlockOperations blockOperations();
    DirectMessageOperations directMessageOperations();
    FriendOperations friendOperations();
    GeoOperations geoOperations();
    ListOperations listOperations();
    SearchOperations searchOperations();
    StreamingOperations streamingOperations();
    TimelineOperations timelineOperations();
    UserOperations userOperations();
    RestOperations restOperations();
}

public interface DirectMessageOperations {
    List<DirectMessage> getDirectMessagesReceived();
    List<DirectMessage> getDirectMessagesReceived(int page, int pageSize);
    List<DirectMessage> getDirectMessagesReceived(int page, int pageSize, long sinceId, long maxId);
    List<DirectMessage> getDirectMessagesSent();
    List<DirectMessage> getDirectMessagesSent(int page, int pageSize);
    List<DirectMessage> getDirectMessagesSent(int page, int pageSize, long sinceId, long maxId);
    DirectMessage getDirectMessage(long id);
    void sendDirectMessage(String toScreenName, String text);
    void sendDirectMessage(long toUserId, String text);
    void deleteDirectMessage(long messageId);
}
```

###实现 Java API binding
API开发者对于Java API binding的实现是完全自由的。Spring Social现有的API Binding实现`spring-social-twitter`使用了`Spring Framework’s RestTemplate`作为REST client；使用`Jackson JSON ObjectMapper`对Json数据进行序列化；使用`Apache HttpComponents`作为http客户端。

Spring Social推荐采用API的实现类命名为"{ProviderId}Template"的惯例。

API binding的实现类的构造方法根据不同的认证协议可能会不同，比如：
OAuth1
```
public TwitterTemplate(String consumerKey, String consumerSecret, String accessToken,
    String accessTokenSecret) { ... }
```
OAuth2
```
public FacebookTemplate(String accessToken) { ... }
```

对API服务器的每个请求中需要用认证证书签名（比如，请求头中带入acess token），这需要在API binding的绑定构建中完成。实际上签名的过程就是，在每个客户端请求执行之前，添加 "Authorization"头到请求的过程。这个签名过程各个协议的不同，实现的方式也不同。为了封装这些复杂性逻辑，Spring Social为每个认证协议提供了`ApiTemplate`基类以供继承。比如，
OAuth1:
```
public class TwitterTemplate extends AbstractOAuth1ApiBinding {
    public TwitterTemplate(String consumerKey, String consumerSecret, String accessToken,
            String accessTokenSecret) {
        super(consumerKey, consumerSecret, accessToken, accessTokenSecret);
    }
}
```
OAuth2：
```
public class FacebookTemplate extends AbstractOAuth2ApiBinding {
    public FacebookTemplate(String accessToken) {
        super(accessToken);
    }
}
```

配置完成后，就可以通过调用`getRestTemplate()`，然后实现不同的API调用。
```
public TwitterProfile getUserProfile() {
    return getRestTemplate().getForObject(buildUri("account/verify_credentials.json"),
        TwitterProfile.class);
}
```

创建`ServiceProvider`模型
------------------------
API bing的过程中需要提供有效的认证证书，证书的获取是通过让应用执行一个认证流程（"authorization dance"）来获取的。`Spring Social`提供`ServiceProvider<A>`抽象来处理"authorization dance"。

由于"authorization dance"是特定于协议的，对于每个认证协议都会有特定的ServiceProvider实现。

###OAuth2
实现基于OAuth2的ServiceProvider，创建`AbstractOAuth2ServiceProvider`的子类，并且命名为{ProviderId}ServiceProvider。

```
public final class FacebookServiceProvider extends AbstractOAuth2ServiceProvider<Facebook> {

    public FacebookServiceProvider(String clientId, String clientSecret) {
        super(new OAuth2Template(clientId, clientSecret,
            "https://graph.facebook.com/oauth/authorize",
            "https://graph.facebook.com/oauth/access_token"));
    }

    public Facebook getApi(String accessToken) {
        return new FacebookTemplate(accessToken);
    }

}
```
在构造方法中，需要调用super方法，传入实现了`OAuth2Operations`的`OAuth2Template`。实际是由`OAuth2Template`来处理
"OAuth dance"的。


创建ApiAdapter
-----------------------
`Connection`的职责是一是提供连接的用户账户的通用的，跨service providers的抽象。`ApiAdapter`的角色则是映射provider的本地API接口到统一的`Connection`模型上。`connection`代理给`ApiAdapter`来执行操作，例如，验证API证书，设置元数据等等。
```
public interface ApiAdapter<A> {
    boolean test(A api);
    void setConnectionValues(A api, ConnectionValues values);
    UserProfile fetchUserProfile(A api);
    void updateStatus(A api, String message);
}
```
`org.springframework.social.twitter.connect.TwitterAdapter`是它的一个实现：
```
public class TwitterAdapter implements ApiAdapter<Twitter> {

    public boolean test(Twitter twitter) {
        try {
            twitter.userOperations().getUserProfile();
            return true;
        } catch (ApiException e) {
            return false;
        }
    }

    public void setConnectionValues(Twitter twitter, ConnectionValues values) {
        TwitterProfile profile = twitter.userOperations().getUserProfile();
        values.setProviderUserId(Long.toString(profile.getId()));
        values.setDisplayName("@" + profile.getScreenName());
        values.setProfileUrl(profile.getProfileUrl());
        values.setImageUrl(profile.getProfileImageUrl());
    }

    public UserProfile fetchUserProfile(Twitter twitter) {
        TwitterProfile profile = twitter.userOperations().getUserProfile();
        return new UserProfileBuilder().setName(profile.getName()).setUsername(
            profile.getScreenName()).build();
    }

    public void updateStatus(Twitter twitter, String message) {
        twitter.timelineOperations().updateStatus(message);
    }

}
```

创建ConnectionFactory
---------------------
现在，已经完成了provider’s API的API binding，ServiceProvider实现了"authorization dance"，ApiAdapter实现了统一`Connection`模型的映射。最后需要创建`ConnectionFactory`用来包装这些东西，并且提供一个简单的接口来建立`Connection`。

类似`ServiceProvider`，`ConnectionFactory`对于不同的协议也有不同的实现。

###OAuth2
创建`OAuth2ConnectionFactory`的子类命名为{ProviderId}ConnectionFactory。在构造方法中调用super，传入`providerId`，{ProviderId}ServiceProvider实例以及{Provider}Adapter实例
```
public class FacebookConnectionFactory extends OAuth2ConnectionFactory<Facebook> {
    public FacebookConnectionFactory(String clientId, String clientSecret) {
        super("facebook", new FacebookServiceProvider(clientId, clientSecret), new FacebookAdapter());
    }
}
```

测试Java API binding
--------------------
这里进行测试的主要目的是保证调用API接口方法是，进行了正确的绑定了，请求了正确了URL，设置了正确的请求参数，服务器返回的数据得到了正确的转换。

这里需要引入类`MockRestServiceServer`:
```
TwitterTemplate twitter = new TwitterTemplate("consumerKey", "consumerSecret", "accessToken",
    "accessTokenSecret");
MockRestServer mockServer = MockRestServiceServer.createServer(twitter.getRestTemplate());
```
`MockRestServiceServer`会记录API Binding的请求，并提供了方法来验证。

```
@Test
public void getUserProfile() {
    HttpHeaders responseHeaders = new HttpHeaders();
    responseHeaders.setContentType(MediaType.APPLICATION_JSON);

    mockServer.expect(requestTo("https://api.twitter.com/1.1/account/verify_credentials.json"))
              .andExpect(method(GET))
              .andRespond(withSuccess(jsonResource("twitter-profile"), APPLICATION_JSON));

    TwitterProfile profile = twitter.userOperations().getUserProfile();
    assertEquals(161064614, profile.getId());
    assertEquals("jbauer", profile.getScreenName());
}
```


扩展ClientHttpRequestInterceptor
--------------------------------
`RestTemplate`可以通过配置拦截器，对请求进行拦截、包装`HttpRequest`，对请求封装一些共同的逻辑，例如，API的Accept头中设置
MediaType或者API Version头等等。拦截器需要实现`ClientHttpRequestInterceptor`接口方法：

```
ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException;
```

其中
- `HttpRequestWrappers`是`HttpRequest`实现类的包装类
- `HttpRequestDecorator`继承自`HttpRequestWrappers`，是个装饰类，可以利用装饰者模式对`HttpRequest`进行扩展。例如：
        
        @Override
        public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
            HttpRequest protectedResourceRequest = new HttpRequestDecorator(request);
            protectedResourceRequest.getHeaders().add("Accept", mediaType.toString());
            return execution.execute(new HttpRequestDecorator(request) {

                @Override
                public URI getURI() {
                    URI uri = super.getURI();
                    URI baseUri = null;
                    try {
                        baseUri = new URI(URL_API);
                    } catch (URISyntaxException e) {
                    }

                    return baseUri.resolve(uri);

                }
            }, body);
        }

总结
====
以OAuth2的为例
- 应用实例化`OAuth2ConnectionFactory`，`ConnectionFactory`是对API binding，ServiceProvider，ApiAdapter的包装
- `ConnectionFactory`调用`getOAuthOperation`获取接口，用于进行OAuth2授权
- `getOAuthOperation`返回的实例实际是由ServiceProvider提供的，ServiceProvider负责"authorization dance"，框架提供了`AbstractOAuth2ServiceProvider`实现OAuth2的验证流程
- 验证流程最终的目的是获取`GrantAccess`，用户建立`Connection`
- 传递`GrantAccess`给`OAuth2ConnectionFactory#createConnection`创建`Connection`
- `Connection`接口提供了一些接口获取远程用户的元数据信息，实际是代理给`ApiAdapter`来实现调用的，目的是为了保持`Connection`统一的接口模型
- `Connection#getApi`获取API binding，供应用调用远程接口。
- API binding的实现最好接口和实现分离，接口最好分层定义。接口的实现根据授权协议的不同，继承不同的基类，OAuth2继承`AbstractOAuth2ApiBinding`，基类主要是作用是能够返回配置好的RestTemplate以供实现进行远程调用，配置RestTemplate主要是要根据授权协议，设置好请求头的参数。

[1]: http://www.rfcreader.com/#rfc6749