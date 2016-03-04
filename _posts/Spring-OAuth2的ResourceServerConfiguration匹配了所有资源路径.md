title: Spring OAuth2的ResourceServerConfiguration匹配了所有资源路径
date: 2016-03-04 13:35:24
tags:
- Spring Security
- Spring OAuth
- 经验总结
---

## 问题

通过继承`ResourceServerConfigurerAdapter`来配置ResourceServer的资源权限时，虽然使用了`HttpSecurity#requestMatchers`方法来匹配特定的资源，但是其他未匹配的资源依然会被匹配保护。

github上同样的[问题](https://github.com/spring-projects/spring-security-oauth/issues/444)。

<!-- more -->

## 原因

这在使用spring-security-oauth的2.0.9之前的版本配合Spring Security 4时会出现这个问题。原因是Spring Security 4.x版本稍微修改了request matchers的规则。多次调用`HttpSecurity#requestMatchers`不会覆盖之前的调用。 2.0.9做了如下[修改](https://github.com/spring-projects/spring-security-oauth/commit/21fce68a2a28794e7627f8e25500e760b352e349)


```
if (endpoints != null) {
  			// Assume we are in an Authorization Server
 -			requests.requestMatchers(new NotOAuthRequestMatcher(endpoints
 -					.oauth2EndpointHandlerMapping()));
 +			http.requestMatcher(new NotOAuthRequestMatcher(endpoints.oauth2EndpointHandlerMapping()));
}	
```

## 解决

Spring Security 4需要使用Spring Security OAuth 2.0.9及以上的版本，如果使用Spring Boot，则需要使用1.3.3级以上版本，1.3.2依赖的依然是Spring Security OAuth 2.0.8版本。