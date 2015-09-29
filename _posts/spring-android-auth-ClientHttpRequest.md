title: spring-android-auth ClientHttpRequest
date: 2015-02-04 21:56:48
tags:
- Spring Framework
- Android
- 经验错误
---

问题
====
使用spring-android-auth时，其使用的`RestTemplate`是依赖包spring-android-rest-template`中的，而是spring-web-mvc中的。该`RestTemplate`使用http request默认的逻辑如下

```
private static final boolean HTTP_COMPONENTS_AVAILABLE = ClassUtils.isPresent("org.apache.http.client.HttpClient", ClientHttpRequestFactory.class.getClassLoader());

if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.GINGERBREAD) {
	this.requestFactory = new SimpleClientHttpRequestFactory();
} else {
	this.requestFactory = new HttpComponentsClientHttpRequestFactory();
}
```
<!--more-->

使用spring-android-auth的主要目的是使用其依赖的spring-social，其中的`OAuth1Template`,`OAuth2Template`等使用的RestTempalte中使用的http request是`ClientHttpRequestFactory`。

```
public static ClientHttpRequestFactory getRequestFactory() {
	Properties properties = System.getProperties();
	String proxyHost = properties.getProperty("http.proxyHost");
	int proxyPort = properties.containsKey("http.proxyPort") ? Integer.valueOf(properties.getProperty("http.proxyPort")) : 80;
	if (HTTP_COMPONENTS_AVAILABLE) {
		return HttpComponentsClientRequestFactoryCreator.createRequestFactory(proxyHost, proxyPort);
	} else {
		SimpleClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();
		if (proxyHost != null) {
			requestFactory.setProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress(proxyHost, proxyPort)));
		}
		return requestFactory;
	}
}
```
由于Android SDK中带有`HttpClient`这个类的，但是又不包含创建的实例对象CloseableHttpClient，实际会报错ClassNotFound。

解决
====
spring-android的构建脚本中对于spring-android-rest-template模块的依赖

```
project("spring-android-rest-template") {
	description = "Spring for Android Rest Template"

	ext.mavenTestDir = "../test/spring-android-rest-template-test/pom.xml"

	dependencies {
		provided("com.google.android:android:$androidVersion")
		compile(project(":spring-android-core"))
		optional("org.apache.httpcomponents:httpclient-android:$httpclientVersion")
		optional("com.squareup.okhttp:okhttp-urlconnection:$okHttpVersion")
		optional("org.codehaus.jackson:jackson-mapper-asl:$jacksonVersion")
		optional("com.fasterxml.jackson.core:jackson-databind:$jackson2Version")
		optional("com.google.code.gson:gson:$gsonVersion")
		optional("org.simpleframework:simple-xml:$simpleXmlVersion") { dep ->
			transitive = false
		}
	}
}
```
httpclient-android是一个optional的依赖，所以使用HttpClient需要显示的指定依赖。

```
compile group: 'org.apache.httpcomponents', name: 'httpclient-android', version:'4.3.3'
```


[相关issue][1]

[1]: https://github.com/spring-projects/spring-social/issues/111