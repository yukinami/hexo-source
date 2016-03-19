title: 如何高效的使用postman
date: 2016-03-19 13:59:46
tags:
- API
- POSTMAN
---

POSTMAN目前在我们团队中一直作为一个调用API的测试工具，但是由于我们的一直深入的使用它，导致使用的过程中存在不少很低效的问题。POSTMAN本身提供很多强大的功能，用好了它完全可以成为开发过程中比不可少的开发工具。

<!-- more -->

## 将POSTMAN作为API文档

我认为一个非常完善的RESTful API，应该提供下面的这些特性。

1. HATEOAS [这里](https://spring.io/understanding/HATEOAS)可以做个简单的直观的了解。有了HATEOAS的支持，客户端不需要去查看API文档就能够知道如何调用接口，也不需要自己去拼接请求URL。而且通常只有很好地实现了RESTful的接口，才能实现HATEOAS，还是设计良好的接口的体现。
2. ALPS ALPS可以说的接口IO的META描述。通过它可以更准确的描述接口。
  > ALPS is a data format for defining simple descriptions of application-level semantics, similar in complexity to HTML microformats. An ALPS document can be used as a profile to explain the application semantics of a document with an application-agnostic media type (such as HTML, HAL, Collection+JSON, Siren, etc.). This increases the reusability of profile documents across media types.

这是我们API实现的一个目标，但是在上述的内容没有很好的被实现的情况下，提供API文档是很必要的。在之前我尝试过使用[swagger](http://swagger.io/)来描述接口，但是swagger的描述文件的书写比较繁琐，第三方的自动生成swagger描述文件的springfox以及swagger-ui一直有不少的问题，达不到一个理想的状态。

因此我们不妨直接使用POST来作为API的文档，一个好的文档不仅要有接口的文字描述，同时还可以提供UI进行直接调用，POSTMAN完全满足这些需求。 另外还好一个好处是，这个文档书写几乎是没有任何的额外成本的，因为开发人员本身也需要POSTMAN来调试接口。让开发人员来书写这个文档，既准确又省时省力。

另外POSTMAN提供了导入导出以及Shared的功能，即便不适用Team相关的功能，也能够在团队中较好的分享分档。

## POST使用中的一些问题

### 环境的问题

开发人员开发完的接口文档是的接口URL地址通常是指向本地的localhost环境的，但是提供给其他团队成员的接口通常是需要调用另外一台服务器的，其他成员不得不再次去修改URL的部分内容，非常繁琐。

#### 解决方法

这个问题可以通过POSTMAN的Environment的功能来解决。因为两边的调用接口不同的只是HOSTNAME和端口，因此可以把这部分定义为环境变量。


![postman-environment-define](/img/postman-environment-define.png)

然后在其他地方引用即可，POSTMAN中引用环境变量的方式是`{{variableName}}`。

![postman-environment-usage](/img/postman-environment-usage.png)


类似的，不同环境下的用户名密码也可以通过这种方式来解决。

### Token的问题

POSTMAN的验证方式中并不提供OAuth2的Password验证方式，这也给我们造成了不少麻烦。我们不得不每次先得请求Token的接口，然后复制下返回的Token值，然后粘贴到`Authorization`头中。这个动作其实相当的低效，特别是在开发环境需要不停的重启服务器的情况下。

#### 解决方法

POSTMAN提供了Pre-request Scirpt和Tests这两个功能，分别在请求之前和请求之后，利用代码进行一些扩展或者额外的操作。

这里我们可以通过在请求Token的接口中，定义如下Tests脚本

```
var data = JSON.parse(responseBody);
postman.setEnvironmentVariable("accessToken", data.access_token);
postman.setEnvironmentVariable("refreshToken", data.refresh_token);
```

然后每个请求的`Authorization`头中直接应用这个环境变量，就可以省去复制粘贴的操作了。

![authorization-token](/img/authorization-token.png)

## 其他一些功能的使用

### Tests

Tests功能本身的设计使用来进行测试的，其中可以书写类似下面的断言

```
tests[“Body contains user_id”] = responseBody.has(“user_id”) 
```

然后在POSTMAN中可以查看结果

![postman-tests-result](https://www.getpostman.com/img/v1/docs/source/cr-6.png)

甚至可以通过Runner来批量的跑接口测试，实现E2E的回归测试等。

![postman-runner](/img/postman-runner.png)

### Generate Code

POSTMAN还提供给了一个小功能，能够把POSTMAN中的请求，转换成其他语言的代码，方便在代码中集成或者调试。

![post-man-generate-code](/img/post-man-generate-code.png)


