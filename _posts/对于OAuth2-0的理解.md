title: 对于OAuth2.0授权方式的理解
date: 2015-01-06 21:23:21
tags:
 - OAuth2
---

概述
====
OAuth2.0的最好的文档，莫过于[RFC 6749][RFC6749]。OAuth2.0本身是一个比较灵活的标准，能够适配应用到各种场景。本文主要是对文档的翻译以及加上部分个人的理解。

角色
====
OAuth定义了4个角色
- resource owner
    能够对受保护的资源进行授权的实体。当`resource owner`是人的时候，它是指终端用户
- resource server
    持有受保护的资源的服务器，使用`access tokens`能够接收和响应对受保护资源的请求
- client
    在resource `owner`的行为和授权下，请求受保护资源的应用。“client”没有特指任何实现特性（比如，无论应用执行在服务器，桌面或者其他设备上）
- authorization server
    当成功认证了`resource owner`并且获得授权后，分发`access tokens`给`client`的服务器。

<!--more-->

Client类型
==========
OAuth定义了两种`client`类型，根据它们与`authorization server`安全地授权的能力（维护`client credentials`的保密性的能力）。
- confidential
    能够维护它们的`credentials`的保密性（例如，对`client credentials`的访问进行限制的安全的服务器上实现的`client`）的`client`。具体的应用类型：web application

- public
    不能够维护它们的`credentials`的保密性的`client`（例如，执行在`resource owner`的设备上的`client`，比如安装的本地应用或者基于浏览器的web应用）。具体的应用类型：user-agent-based application， native application

协议流程
=======

     +--------+                               +---------------+
     |        |--(A)- Authorization Request ->|   Resource    |
     |        |                               |     Owner     |
     |        |<-(B)-- Authorization Grant ---|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(C)-- Authorization Grant -->| Authorization |
     | Client |                               |     Server    |
     |        |<-(D)----- Access Token -------|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(E)----- Access Token ------>|    Resource   |
     |        |                               |     Server    |
     |        |<-(F)--- Protected Resource ---|               |
     +--------+                               +---------------+

(A) `client`从`resource owner`请求认证。认证请求可以直接向认证服务器请求或者间接地通过`authorization server`
(B) `client`收到一个`authorization grant`，实际是代表了`resource owner`的认证的证书，使用文档中定义的四种授权类型或者扩展的授权类型来表示
(C) `client`向`authorization server`提交`authorization grant`，请求授权获取`access token`，
(D) `authorization server`授权`client`，验证`authorization grant`，如果是有效的，分发`access token`
(E) `client`从`resource server`提交`access token`请求受保护的资源
(F) `resource server`验证`access token`，如果是有效的，接收请求

授权模式（Authorization Grant）
=============================
对于协议的流程来说，最灵活的部分就是上面提到的(B)中`Authorization Grant`。也就是说(C)请求授权`access token`所提交的`authorization grant`，根据授权模式不同是不一样的。

[RFC 6749][RFC6749]文档中提及了4种授权模式：
- Authorisation code grant
- Implicit grant
- Resource owner credentials grant
- Client credentials grant

Authorisation Code
------------------------
`authorization code`是通过使用`authorization server`作为`client`和`resource owner`的中间人获取的。`client`把`resource owner`导向到`authorizatoin server`（通过它的用户代理），然后反过来带着`authorization code`把`resource owner`导向回`client`。

在附带着`authorization code`把`resource owner`导向回`client`之前，`authorization server`认证`resource owner`并且获取授权。 因为`resource owner`只通过`authorization server`进行验证，`resource owner`的证书不需要和`client`共享。

`authorization code`提供了一些重要的安全方面的好处，比如说可以验证`client`，并且直接传递`access token`给`client`，不需要经过`resource owner`的用户代理，所以不会潜在地暴露给别人包括`resource owner`本身

下面的图演示整个流程
![oauth2 work flow](/img/oauth2_flow.png)

Implicit
--------
`implicit`授权为在浏览器中使用脚本语言（Javascript）实现的客户端而优化的简单版的`authorization code`流程。在`implicit`的流程中，`client`直接请求一个`access token`(作为`resource owner`授权的结果)而不是为`client`请求一个`authorization code`

在`implicit`的流程中请求一个`access token`时，`authorization server`不验证客户端。在一些情况下，`client`的标示，可以通过，用来传递`access token`给`client`的`redirection URI`来验证。`access token`会暴露给`resource owner`或者其他访问`resource owner`的用户代理的应用程序（因为首先要经过用户代理的重定向才能把`access token`传递给`client`）。

`Implicit`授权提高了响应效率以及一些`client`的效率（比如用浏览应用实现的`client`），因为它减少了为了获取`access token`而进行的来回交互。然而，这些便利应该和安全影响进行权衡。

Resource Owner Password Credentials
------------------------------------
`resource owner password credentials`（比如说用户名密码）， 能够被直接用作`authorization grant`来获取`access token`。`credentials`只应该用在`resource owner`和`client`之间是高度信任的情况下（例如，`client`是设备操作系统的一部分或者一个具有高权限的应用），并且其他的授权方式不可用的情况下。

尽管这种方式需要`client`直接访问`resource owner`的`credentials`，`resource owner`的`credentials`仅被用于一次请求来交换`access token`。这种授权类型能够通过用`credentials`交换长期存活的`access token`或者`refresh token`，排除`client`了为了以后的使用而保存`resource owner`的`credentials`的需要。

Client Credentials
------------------
当授权范围被限制在，`client`控制下的受保护的资源的时候或者，受保护的资源事先和`authorization server`协商好的时候，能够使用`client credentials`（或者其他形式的`client`验证）。当`client`是在以自己名义行动（`client`就是就是`resource owner`），或者根据和`authorization server`事先协商好的授权访问受保护的资源的时候，使用`Client credentials`。

[RFC6749]: http://www.rfcreader.com/#rfc6749

其他
====

Access Token Scope
------------------
`authorization server`允许`client`获取`access token`的时候，指定`scope`参数。 这样获取的`access token`之后只能访问对应的`scope`的资源。资源服务器需要实现对资源的`scope`进行保护。`scope`的参数的值可以是多个用空格间隔的字符串，大写小敏感。`authorization server`需要定于`client`和其可获取的`scope`的对应关系，同时定义一个默认的`scope`。当请求`access token`实际获取的`scope`与`client`请求的`scope`不同的时候，`authorization server`需要返回实际授予的`scope`参数。

实际应用的时候，可以实现一个类似RBAC的简化模型。`authorization server`定义的`scope`相当于定义了资源所属的角色。`client`定义多个需要的`scope`为一个access token type，相当于给用户赋予角色。根据`client`的不同，赋予不同的角色，即可实现对同一用户在不同的`client`中所能访问的资源的权限控制。