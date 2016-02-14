title: spring boot error handling
date: 2016-02-14 15:33:24
tags:
- Spring MVC
- Spring Boot
---

## Spring MVC的异常处理

`HandlerExceptionResolver`的实现用来处理意外的异常。

### ExceptionHandlerExceptionResolver

`ExceptionHandlerExceptionResolver`实现通常寻找@ExceptionHandler注解的方法来处理异常。

```
@Controller
public class SimpleController {

    // @RequestMapping methods omitted ...

    @ExceptionHandler(IOException.class)
    public ResponseEntity<String> handleIOException(IOException ex) {
        // prepare responseEntity
        return responseEntity;
    }

}
```
<!--more-->

当前Contorller抛出@ExceptionHandler注解的异常时，被注解的handleIOException方法会执行，返回对应的结果。返回值可以会被当成view name的String，也可以是ModelAndView、ResponseEntity对象，添加@ResponseBody注解等等。 也可以在@ControllerAdvice注解的Controller内定义全局的@ExceptionHandler方法，让其对所有的Controller生效。


### DefaultHandlerExceptionResolver

`DefaultHandlerExceptionResolver`的实现会翻译标准的Spring MVC的异常为默认的HTTP状态码，但是不会往response body中写入错误内容。

如果想要写入错误内容，可以继承`ResponseEntityExceptionHandler`。这个是@ControllerAdvice的便利的基类，提供了@ExceptionHandler方法来处理标准的Spring MVC异常。

### ResponseStatusExceptionResolver

`ResponseStatusExceptionResolver`的实现，能够根据异常类的@ResponseStatus注解来返回需要的HTTP状态码。

### 自定义Servlet容器的错误页

当Response被设置了错误的状态码，但是当时body为空时，Servlet容器通常会渲染一个默认的HTML错误页。 要自定义这个错误页可以通过web.xml的`<error-page>`元素来设置。在Servlet 3之前，必须映射到特定的状态码或异常类型。从Servlet 3开始，可以直接默认的错误页面。

```
<error-page>
    <location>/error</location>
</error-page>
```

这个错误页面的实际地址可以是JSP页面，也可以是能够被@Controller方法处理的URL。

```
@Controller
public class ErrorController {

    @RequestMapping(path = "/error", produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
    @ResponseBody
    public Map<String, Object> handle(HttpServletRequest request) {

        Map<String, Object> map = new HashMap<String, Object>();
        map.put("status", request.getAttribute("javax.servlet.error.status_code"));
        map.put("reason", request.getAttribute("javax.servlet.error.message"));

        return map;
    }

}
```

## Spring Boot的处理

Spring Boot默认提供了`/error`的映射来处理所有的异常，并且在servlet container中将其注册为了全局的错误页面。

要完全替换默认的行为，可以实现`ErrorController`接口并注册该类型的Bean。也可以仅仅注册`ErrorAttributes`类型的Bean替换部分错误的内容，或者提供一个HTML的error view来替换默认的页面样式(默认的HTML error view定义在`WhitelabelErrorViewConfiguration`中)。