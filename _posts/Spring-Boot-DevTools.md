title: Spring Boot DevTools
date: 2016-01-18 16:26:17
tags:
- Spring Boot
- 热部署
---

Java相对于其他的一些开发语言来说，修改代码后经常需要重新编译代码，重启服务器等等，这些动作非常的耗时间，降低了不少生产效率，所以热部署一直是Java的开发人员所追求的。从JRebel到Spring Loader，它们都可以达到JVM的Hot Swapping的目的。但是针对类似Spring这样需要在服务器启动时初始化一些容器配置的项目，还是不得不把大量的时间花在重启服务器上。

<!--more-->

Spring Boot从1.3开始，引入了一个新的模块叫做`spring-boot-devtools`，目的是节省开发Spring Boot应用时的一些时间，它引入下面的一些特性。


Property Defaults
==================

当我们在使用一些类似Thymeleaf的模板引擎的时候，需要把属性`spring.thymeleaf.cache`设置成`false`来关闭缓存，从而达到不重启服务器更新页面的目的。 现在`spring-boot-devtools`模块自动关闭Thymeleaf, Freemarker, Groovy Templates, Velocity and Mustache这些模板引擎的缓存。

Automatic Restart
==================

Spring Boot 1.3使用了一个叫做“instant reload”的技术，当你修改了类路径下的文件后，能够自动的重启服务器。这虽然比JRebel和Spring Loader这些即时重载技术会稍慢一点，但是它重启服务器的速度非常之快，并且Spring容器相关的配置也会生效。

NOTE: 另外在使用Intelij IDEA作为IDE时，为了让Automatic Restart工作，必须手动触发Make Project更新class文件，因为Automatic Restart是针对的class文件而不是源文件


LiveReload
===========

当使用“cache properties”和“automatic restarts”时，最终还是需要点击浏览器的刷新来重载页面，这显得有点繁琐。Spring Boot 1.3 DevTools包含了内嵌的LiveReload服务器。LiveReload是一个允许你的应用出发浏览器刷新的简单协议。浏览器的扩展在[这里][livereload-extensions]。

Remote Debug Tunneling
=======================

如果你用Docker运行你的应用，那么Debug是非常麻烦的。你必须让Java用`-Xdebug`参数启动，然后才能使用remote debugger。Spring Boot 1.3能够为JDWP (the Java Debug Wire Protocol) 启用HTTP隧道。这能够让我们对只暴露了80和443接口的部署在云上的应用进行debug。

最后Developer tools在运行一个完整打包的应用时会自动关闭。如果你的应用使用`java -jar`运行或者使用一个特殊的classloader启动，那么会被认为是“生产环境”。

原文介绍参考[这里][devtools-in-spring-boot]

[livereload-extensions]:http://livereload.com/extensions/
[devtools-in-spring-boot]:https://spring.io/blog/2015/06/17/devtools-in-spring-boot-1-3