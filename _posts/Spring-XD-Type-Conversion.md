title: Spring XD Type Conversion
date: 2015-01-13 22:12:06
tags: 
- Spring XD
- Spring Ingegration
---

Spring Integration 的 Type Conversion
=====================================

Datatype Channel
-----------------
Datatype Channel 可以指定Channel接收的数据的数据类型
```
<int:channel id="numberChannel" datatype="java.lang.Number"/>
<int:channel id="stringOrNumberChannel" datatype="java.lang.String,java.lang.Number"/>
```
指定了`datatype`之后，如果Channel收到了类型不是指定的类型，会如下进行：
1. 查找是否定义了一个叫做"integrationConversionService"的Bean，该Bean是Spring's [Conversion Service][1]的实例。如果没有，则会抛出异常。
2. 如果存在，则会尝试使用它将Message的Payload类型转换为Channel的`datatype`

<!--more-->

定义了Converter的方式如下：
1. 实现`Converter`接口
		public static class StringToIntegerConverter implements Converter<String, Integer> {
			public Integer convert(String source) {
				return Integer.parseInt(source);
			}
		}
2. Integration Conversion Service中注册为一个Converter
		<int:converter ref="strToInt"/>
		<bean id="strToInt" class="org.springframework.integration.util.Demo.StringToIntegerConverter"/>
当`converter`元素被解析时，容器会注册名为"integrationConversionService"的bean，同时将convert注入到该bean中。

***从4.0版本开始***
`integrationConversionService`会由`DefaultDatatypeChannelMessageConverter`在application conext中查找，并调用。可以通过指定channel的`message-converter`属性，来只用另外的转换机制，指定的值需要实现`MessageConverter`接口。

同时也可以通过注解的方式注册converter：
```
@Component
@IntegrationConverter
public class TestConverter implements Converter<Boolean, Number> {

	public Number convert(Boolean source) {
		return source ? 1 : 0;
	}

}
```

Spring XD 的 Type Conversion
============================
Spring XD 通过模块参数`--outputType=application/json`，让Spring XD注册实例的的时候动态设置datatype的方式，实现type conversion，本质上还是利用了Spring Integration的转换机制来实现的。

Spring XD 环境中使用`CompositeMessageConverter`（`message-converter`属性）来进行转换。Spring XD将`xd.messageConverters`和`customMessageConverters`的两个list注入到`CompositeMessageConverter`，并代理给它们进行实际的转换工作。

之所以`CompositeMessageConverter`的机制是比较满意的，是因为在 Spring XD 环境中，一个模块的从上一个模块的接收到Message的Payload类型是不确定的（Converter的Source类型是不确定的）。比如，Spring XD 提供的转换器能够将任何Java对象转换为`Tuple`。但是针对每个类转换到`Tuple`的方式是不一样的。所以`CompositeMessageConverter`需要根据`outputType`参数以及实际的传入类型来找到合适的转换器进行转换。

在Spring XD中实现自定义的转换器，只需要注册"customMessageConverters"，并添加到converter其中
```
<!-- Users can override this to add converters.-->
<util:list id="customMessageConverters">
    <bean class="..."/>
</util:list>
```

向"customMessageConverters"中注册的converter需要继承`AbstractFromMessageConverter`类。


[1]: http://docs.spring.io/spring/docs/current/spring-framework-reference/html/validation.html#core-convert-ConversionService-API