title: Spring Boot应用从main方法运行
date: 2016-02-15 09:45:37
tags:
- Spring Boot
- Gradle
---

Spring Boot的应用是可以通过main方法中运行SpringApplication.run()来启动的。如果项目使用了gradle或者maven，则可以通过`gradle bootRun`或者`mvn spring-boot:run`等插件来运行。

但是运行起来是有不同的，因为通过gradle或者maven运行，他们运行起来之后能够正确的处理一些runtime和provided的依赖，但是通过main方法运行的cp则是IDE中固定的。

<!--more-->

## 依赖的定义

Maven默认的提供的依赖级别有compile、runtime、provided，后两者分别对于仅仅打包（运行）需要和打包（运行）不需要。 

Gradle本身没有提供provided级别的。Gradle的war插件在这基础上又添加providedCompile、providedRuntime这两个依赖级别，解释是说这两个依赖界别分别对应compile和runtime，除了它们不会添加到war包中。 也就是说war插件把运行和打包这两个阶段区别开了。

总结来说

|                  |    编译      |    运行       |     打包     |
|:-----------------|:------------:|:------------:|:------------:|
| compile          |    是        |    是         |     是      |
| runtime          |    否        |    是         |     是      |
| provided         |    是        |    否         |     否      |
| providedRuntime  |    否        |    是         |     否      |
| providedCompile  |    是        |    是         |     否      |


## 问题

因为项目需要最终打成war包，放在Tomcat中运行。所以对于Tomcat的依赖，项目中这么定义的

```
providedRuntime("org.springframework.boot:spring-boot-starter-tomcat")
```

通过main方法在IDE（IDEA）中运行时，会找不到报找不到Servlet API的错误。

## 原因

这样的话，通过bootRun运行，tomcat能够正确依赖，打包时，tomcat依赖不会被加入到war包中。但是问题是，如果项目导入IDE（以IDEA为例）中，会被解析为provided，也就是说，运行的时候，tomcat依赖会找不到。但是实际对于providedRuntime来说，它就是runtime，只是打包的时候不加入而已。 

## 解决

runtime依赖相较于compile来说只是编译的时候不加入，但是即便runtime修改成compile的话，只是cp中多了几个包而已，也是没有什么副作用的。所以我们即便使用providedCompile也是可以的，但是遗憾的是，IDEA中这个依赖级别也是解析为provided。

我们实际需要的是和compile相同的依赖级别，但是需要从打包中去除，同时又可以让IDEA解析成compile依赖级别的。这可以通过gradle的自定义configuration来实现：

```
configurations {
    compileProvided
}

war {
    classpath = files(configurations.runtime.minus(configurations.compileProvided))
}

dependencies {
    ……
    
    compileProvided("org.springframework.boot:spring-boot-starter-tomcat")
}

```