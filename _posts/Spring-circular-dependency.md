title: Spring circular dependency
date: 2016-12-06 11:22:02
tags:
- Spring Framework
---

当发生类似 `Bean#1 -> Bean#2 -> Bean#1` 的依赖时，Spring框架通常能够自动的进行处理

> You can generally trust Spring to do the right thing. It detects configuration problems, such as references to non-existent beans and circular dependencies, at container load-time. Spring sets properties and resolves dependencies as late as possible, when the bean is actually created.

<!--more-->

如果出现`BeanCurrentlyInCreationException`异常，通常时因为使用了构造方法注入而产生的“鸡生蛋”的问题。解决方法就是使用setter方法注入。

如果使用setter方法依然产生这个问题，例如下面的错误

> Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.BeanCurrentlyInCreationException: Error creating bean with name 'asyncVerifyServiceImpl': Bean with name 'asyncVerifyServiceImpl' has been injected into other beans [verifyServiceImpl] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.

如果循环依赖的中的一个对象最终还会被代理（例如这里包含@Async方法的Bean），就会出这个错。解决方法是在这个Bean上使用@Lazy在延迟这个对象的初始化。