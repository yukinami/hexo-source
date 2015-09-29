title: 同源策略
date: 2015-04-20 10:19:08
tags:
- HTML
---

同源策略
=======

`Same-Origin Policy`根据`origin`限制浏览器通过脚本或者文档执行特定的动作。`origin`指的是URL的path之前的所有的部分(比如说，http://www.example.com )。对于一些动作，浏览器会比较origins，如果它们不匹配，则不运行继续执行。比如说：
- 一个父document不能访问来自不同origin的&lt;iframe&gt;的内容。这可以阻止恶意网站打开你们的银行网站，偷取你的安全秘钥
- 一个document可以通过表单post发送信息，但是跨域AJAX请求通常是不允许的


针对同源策略，有两个方案可以解决这个问题。

<!-- more -->

JSON-P (JSONP)
==============
能够跨域请求的一个机制就是&lt;script&gt;标签。JSON-P通过创建一个&lt;script&gt;元素（通过HTML markup或者使用JavaScript插入DOM），它请求一个远程数据服务地址。它的响应（加载的"JavaScript"的内容）是一个在请求页面预先定义好的函数名，并且包含作为参数的所请求的JSON数据。当脚本一执行，函数被执行并且传入JSON数据，通没这种机制，允许请求页面获取并且处理数据。

例子
```
function handle_data(data) {
   // `data` is now the object representation of the JSON data
}


---
http://some.tld/web/service?callback=handle_data:
---
handle_data({"data_1": "hello world", "data_2": ["the","sun","is","shining"]});
```

问题
----
目前，JSON-P实际上只是约定而成的松散定义，实际上浏览器接收任意的响应JavaSciprt。这意味着依赖于JSON-P来进行跨域进行的作者把他们自己暴露于同源策略起初要解决的问题。

CORS
====

`CORS` (Cross Origin Resource Sharing) 是一个浏览器规范，它允许Javascript代码受控的访问和Javascirpt代码自身不属于同一个源的地址。

工作流程大致如下：
- 一段JavaScript代码（来自源地址 http://a.com ）通过viaXMLHttpRequest通过HTTP请求http://b.com
- 由于脚本的源和目标URL的源不同，浏览器对请求的响应添加了一些额外的在检查
- 对b.com的请求包含`Origin`头： http://a.com
- b.com的服务器的请求响应根据请求的header头来决定是否允许访问。服务器的决定结果包含的响应头`Access-Control-Allow-Origin`中
- 响应头的值可以是一个实际的URL（例如http://a.com）或者通配符，像`Access-Control-Allow-Origin: * `，它允许来自任何源地址的请求
- 浏览器根据返回的访问控制头最终决定返回响应给JavasScript代码。如果不允许，则在处理响应数据前抛出一个异常


上述流程适用于`简单请求`，一个`简单请求`必须满足以下条件：
- HTTP方法必须是GET、HEAD或者POST之一
- 请求必须只包含下面的头
    - Accept
    - Accept-Language 或 Content-Language
    - Content-Type 的值是 application/x-www-form-urlencoded, multipart/form-data 或 text/plain

如果请求不满足标准，那么一个叫做`preflight request`的请求会在实际的请求发生之前先被发送到服务器端。

`preflight`是一个HTTP请求，它的HTTP方法是OPTIONS并且包含`Origin`头（这里是http://a.com），`Access-Control-Request-Method`包含实际请求的HTTP方法以及` Access-Control-Request-Headers`包含实际请求的请求头的名称

潜在的危害。例如，一个意义的web服务能够通过JSONP返回一个函数调用，但是悄悄插入一段JavaScript逻辑到页面中窃取用于的个人数据。