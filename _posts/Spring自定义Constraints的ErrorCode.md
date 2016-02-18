title: Spring自定义Constraints的ErrorCode
date: 2016-02-18 17:40:31
tags:
- Spring Framework
---

如果使用自定义Constraints，ErrorCode的是根据Constraints的名称按照优先度生成的`{Constraints名}.{类名}.{字段名}`、`{Constraints名}.{字段名}`、`{Constraints名}.{字段类型}`、`{Constraints名}`。 在注解中定义的message会优先给Bean Validation解析，然后使用解析的结果作为Default Message，ErrorCode作为key，到Spring的MessageSource中再进行一次解析，获得最终的message template。

另外被验证字段本身的名称，再加上注解中除了`message`，`groups`，`payload`属性定义的其他属性的值，会作为key再次到MessageSource进行解析，结果的结果作为变量传入到上面的message template中。