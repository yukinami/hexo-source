title: 使用springfox生成springmvc项目的swagger的文档
date: 2015-07-07 13:29:20
tags:
- Spring Framework
- API
- Swagger
---

介绍
====

Swagger
--------
`Swagger`是用来描述和文档化RESTful API的一个项目。[Swagger Spec][1]是一套规范，定义了该如何去描述一个`RESTful API`。类似的项目还有`RAML`、`API Blueprint`。 根据`Swagger Spec`来描述RESTful API的文件称之为`Swagger specification file`，它使用JSON来表述，也支持作为JSON支持的YAML。

<!--more-->

`Swagger specification file`可以用来给[swagger-ui][2]生成一个Web的可交互的文档页面，以可以用[swagger2markup][3]生成静态文档，也可用使用[swagger-codegen][4]生成客户端代码。总之有了有个描述API的JSON文档之后，可以做各种扩展。

`Swagger specification file`可以手动编写，[swagger-editor][5]为了手动编写的工具提供了预览的功能。但是实际写起来也是非常麻烦的，同时还得保持代码和文档的两边同步。于是针对各种语言的各种框架都有一些开源的实现来辅助自动生成这个``Swagger specification file`。

`swagger-core`是一个Java的实现，现在支持`JAX-RS`。`swagger-annotation`定义了一套注解给用户用来描述API。
`spring-fox`也是一个Java的实现，它支持`Spring MVC`， 它也支持`swagger-annotation`定义的部分注解。


使用Springfox
=============

Docket
-------
Springfox通过定义`Docket`对象来全局的定义API的一些属性。同时支持`swagger-annotation`的部分注解来定义各个API方法一些参数等。
看一个定义的例子

```
@SpringBootApplication
@EnableSwagger2
@ComponentScan(basePackageClasses = {
    PetController.class
})
public class Swagger2SpringBoot {

  public static void main(String[] args) {
    ApplicationContext ctx = SpringApplication.run(Swagger2SpringBoot.class, args);
  }


  @Bean
  public Docket petApi() {
    return new Docket(DocumentationType.SWAGGER_2)
        .select() //1
          .apis(RequestHandlerSelectors.any())
          .paths(PathSelectors.any())
          .build()
        .pathMapping("/") //2
        .directModelSubstitute(LocalDate.class, //3
            String.class)
        .genericModelSubstitutes(ResponseEntity.class) //4
        .alternateTypeRules( //5
            newRule(typeResolver.resolve(DeferredResult.class,
                    typeResolver.resolve(ResponseEntity.class, WildcardType.class)),
                typeResolver.resolve(WildcardType.class)))
        .useDefaultResponseMessages(false) //6
        .globalResponseMessage(RequestMethod.GET, //7
            newArrayList(new ResponseMessageBuilder()
                .code(500)
                .message("500 message")
                .responseModel(new ModelRef("Error"))
                .build()))
        .securitySchemes(newArrayList(apiKey())) //8
        .securityContexts(newArrayList(securityContext())) //9
        ;
  }

  @Autowired
  private TypeResolver typeResolver;

  private ApiKey apiKey() {
    return new ApiKey("mykey", "api_key", "header");
  }

  private SecurityContext securityContext() {
    return SecurityContext.builder()
        .securityReferences(defaultAuth())
        .forPaths(PathSelectors.regex("/anyPath.*"))
        .build();
  }

  List<SecurityReference> defaultAuth() {
    AuthorizationScope authorizationScope
        = new AuthorizationScope("global", "accessEverything");
    AuthorizationScope[] authorizationScopes = new AuthorizationScope[1];
    authorizationScopes[0] = authorizationScope;
    return newArrayList(
        new SecurityReference("mykey", authorizationScopes));
  }

  @Bean
  SecurityConfiguration security() {
    return new SecurityConfiguration(
        "test-app-client-id",
        "test-app-realm",
        "test-app",
        "apiKey");
  }

  @Bean
  UiConfiguration uiConfig() {
    return new UiConfiguration(
        "validatorUrl");
  }
}
```


1. 定义了需要生成API文档的endpoint，api()方法可以通过`RequestHandlerSelectors`的各种选择器来选择，比如说选择所有注解了`@RsestController`的类中的所有API e.g. `.apis(RequestHandlerSelectors.withClassAnnotation(RestController.class))`。path()方法可以通过`PathSelectors`的来匹配路径，提供了`regex`匹配或者`ant`匹配
2. 定义了API的根路径
3. 输出模型定义时的替换，比如遇到所有`LocalDate`的字段时，输出成`String`
4. 遇到对应泛型类型的外围类，直接解析成泛型类型，比如说`ResponseEntity<T>`，应该直接输出成类型T
5. 提供了自定义性更强的针对泛型的处理，示例中的代码的意思是将类型DeferredResult<ResponseEntity<T>>直接解析成类型T
6. 是否使用默认的ResponseMessage， 框架默认定义了一些针对各个HTTP方法时各种不同响应值对应的message
7. 全局的定义ResponseMessage，示例代码定义GET方法的500错误的消息以及错误模型。注意这里GET方法的所有ResponseMessage都会被这里的定义覆盖
8. 定义API支持的SecurityScheme，指的是认证方式，支持OAuth、APIkey。 P.S. 要让swagger-ui的oauth正常工作，需要定义个`SecurityConfiguration`的Bean
9. 定义具体上下文路径对应的认证方式
10. 还有一些接口可以定义API的名称等一些基本信息，定义API支持的数据格式等等。


注解
----
还有一些`swagger-annotation`的注解可以帮助我们针对各个API生成更为详细的文档

- @ApiOperation 描述方法名称，描述
- @ApiResponses 描述方法发生异常时返回的code以及message
- @ApiImplicitParams 显示的描述方法的参数，默认情况下springfox会根据参数的`@RequestParam`、`@PathParam`等注解来自动获取参数。 但是如果没有注解则会识别为`request body`。 方法的一些参数是由容器注入的，并不是客户端的参数。使用` @ApiIgnore`来忽略。 


一些坑
------
###swagger-ui的o2c.hmtl找不到
2.0.4版本之前，Oauth2认证返回的redirectUri为`/o2c.html`，但是o2c.html在webjar里面，实际路径是`/webjars/springfox-swagger-ui/o2c.html`，具体[参考][6]

###swagger-ui的oauth认证弹出框失效
2.0.4版本之前存在的问题，之前提到了需要在容易中定义一个`SecurityConfiguration`的Bean，因为`swagger-ui.html`会在初始化的时候请求这个Bean，访问返回对象中的几个属性，然后初始Oauth的一些JS代码。如果后台返回的JSON的property名不是驼峰式命名的，那这几个属性就不能正确获取到，导出初始化失败，具体[参考][7]

静态文档
=======
静态文档可以通过`Swagger2MarkupResultHandler`执行生成asciidoc或者`SwaggerResultHandler`生成`swagger.json`，再使用`swagger2markup-gradle-plugin`根据`swagger.json`生成asciidoc，ResultHandler的使用要配合`Spring Test Framework`。

最后使用`asciidoctor-gradle-plugin`合并asciidoc并生成html




[1]: https://github.com/swagger-api/swagger-spec/blob/master/versions/2.0.md
[2]: https://github.com/swagger-api/swagger-ui
[3]: https://github.com/Swagger2Markup/swagger2markup
[4]: https://github.com/swagger-api/swagger-codegen
[5]: https://github.com/swagger-api/swagger-editor
[6]: https://github.com/springfox/springfox/issues/829
[7]: https://github.com/springfox/springfox/issues/833