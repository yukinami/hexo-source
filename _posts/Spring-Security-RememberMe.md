title: Spring Security RememberMe
date: 2016-02-14 14:25:55
tags:
- Spring Security
---

## Remember-Me接口和实现

```
Authentication autoLogin(HttpServletRequest request, HttpServletResponse response);

void loginFail(HttpServletRequest request, HttpServletResponse response);

void loginSuccess(HttpServletRequest request, HttpServletResponse response,
	Authentication successfulAuthentication);
```
<!--more-->

Remember-me是和`UsernamePasswordAuthenticationFilter`一起使用的，通过其父类`AbstractAuthenticationProcessingFilter`中的钩子来触发回调来实现用户信息的保存。


### TokenBasedRememberMeServices

这个实现是通过把用户名和密码进行哈希，然后将生成token值写入Cookies。Cookie的key值同时会被`authentication provider`共享，以用来进行登录的验证。


### PersistentTokenBasedRememberMeServices

这个实现使用起来是和`TokenBasedRememberMeServices`类似的，只是他不是根据用户名和密码进行哈希，而是生成一个随机的值，然后通过`PersistentTokenRepository`保存起来。

## 配置Remember-Me

Remember-Me的请求参数，生效时间等等都是在`AbstractRememberMeServices`中定义的，通过`RememberMeConfigurer`来配置Remember-Me。

Remember-Me默认的请求参数是"remember-me"，有效时间两周。

```
public static final String DEFAULT_PARAMETER = "remember-me";
public static final int TWO_WEEKS_S = 1209600;
```