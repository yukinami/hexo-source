title: Spring Security CSRF
date: 2016-02-14 14:54:43
tags:
- Spring Security
---

Spring Security可以保护应用免收Cross Site Request Forgery (CSRF)的攻击。

## 什么是CSRF

假设你的银行网站提供如下的表单进行转账

```
POST /transfer HTTP/1.1
Host: bank.example.com
Cookie: JSESSIONID=randomid; Domain=bank.example.com; Secure; HttpOnly
Content-Type: application/x-www-form-urlencoded

amount=100.00&routingNumber=1234&account=9876
```
<!--more-->

现在假设你登陆了银行的网站，然后没有登出，访问了一个恶意网站，恶意网站有这么一个表单

```
<form action="https://bank.example.com/transfer" method="post">
<input type="hidden"
	name="amount"
	value="100.00"/>
<input type="hidden"
	name="routingNumber"
	value="evilsRoutingNumber"/>
<input type="hidden"
	name="account"
	value="evilsAccountNumber"/>
<input type="submit"
	value="Win Money!"/>
</form>
```

并且你点了这个表单，尽管恶意网站不知道你的用户名密码，看不到的cookies，但是最终请求还是被发出，钱就被转走了。并且这还能通过Javascript脚本来实现，就是说恶意网站根本就不需要你点击这个按钮，只要访问了他们的网站就能实现。

### 解决办法

解决的方法就是使用Synchronizer Token Pattern。这个方案确保每个请求，除了session的cookies值，还需要发送一个预先生成的随机值作为请求参数。

```
POST /transfer HTTP/1.1
Host: bank.example.com
Cookie: JSESSIONID=randomid; Domain=bank.example.com; Secure; HttpOnly
Content-Type: application/x-www-form-urlencoded

amount=100.00&routingNumber=1234&account=9876&_csrf=<secure-random>
```

### Spring Security中启用CSRF保护  

#### 配置CSRF保护

Spring Security 4.0，CSRF保护是默认启用的，如果要关闭

```
<http>
	<!-- ... -->
	<csrf disabled="true"/>
</http>
```

```
@EnableWebSecurity
public class WebSecurityConfig extends
WebSecurityConfigurerAdapter {

@Override
protected void configure(HttpSecurity http) throws Exception {
	http
	.csrf().disable();
}
}
```

#### 包含CSRF Token

启用CSRF保护后，需要在请求中包含CSRF Token。下面是一个JSP的例子

```
<c:url var="logoutUrl" value="/logout"/>
<form action="${logoutUrl}"
	method="post">
<input type="submit"
	value="Log out" />
<input type="hidden"
	name="${_csrf.parameterName}"
	value="${_csrf.token}"/>
</form>
```

AJAX的例子

```
<html>
<head>
	<meta name="_csrf" content="${_csrf.token}"/>
	<!-- default header name is X-CSRF-TOKEN -->
	<meta name="_csrf_header" content="${_csrf.headerName}"/>
	<!-- ... -->
</head>
<!-- ... -->

$(function () {
var token = $("meta[name='_csrf']").attr("content");
var header = $("meta[name='_csrf_header']").attr("content");
$(document).ajaxSend(function(e, xhr, options) {
	xhr.setRequestHeader(header, token);
});
});
```

如果使用JSP标签<form:form>，或者Thymeleaf 2.1+，CSRF会自动通过`RequestDataValueProcessor`的子类`CsrfRequestDataValueProcessor`自动加入到form表单中。 前提是需要启用@EnableWebMvcSecurity来生成`CsrfRequestDataValueProcessor`的实例

NOTE: 从Spring Security 4.0开始，@EnableWebMvcSecurity注解被废弃，使用@EnableWebSecurity注解即可。


