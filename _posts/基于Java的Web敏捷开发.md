title: 基于Java的Web敏捷开发
date: 2015-08-03 21:48:49
tags:
- Agile
- Spring Boot
---

敏捷的Web框架，应该至少做到开箱即用，开发时不依赖任何外部数据库和外部的服务器是最理想的。

Web框架
-------
Spring Boot基于Spring平台，涉及到了构建一个Java Web项目的大部分方面，是一个非常好的选择。只要简单的声明依赖，便可支持基于Spring各种特性。Web层基于Spring MVC，持久层支持JPA，Spring Data JPA，View层支持Freemarker等模板语言，支持Message、Application properties，支持日志，支持多个环境的Profile，安全方面支持Spring Security等等。Spring Boot把Spring的各个项目无缝的结合在一起，可以统一的使用Application properties对它们进行配置。

<!--more-->

Spring Boot大大地简化了Java Web项目的开发，不需要像传统的项目那样进行大量的XML或者Java注解的配置。只需要像类似rails一样，在配置文件配置一些参数即可。

[Spring Initializr][spring-initializr]可以用来帮助初始化构建基于Spring Boot的应用。

多Profile的支持
--------------
我们在开发环境和生产环境的时候，项目会连接不同的数据库，使用不同的参数等等，所以通常需要应用支持不同的profile。

通过`spring.profiles.active`属性激活特定的profile、以profile为单位配置应用参数。例如可以这样：

- application.properties
- application-dev.properties
- application-prod.properties
- application-test.properties

多个profile共通的参数配置在application.properties

数据库
-----
开发时应该使用一个内存数据库，一方面便于开发环境的构建，不需要连接特定的外部数据库，可以在任何网络环境下进行开发；另一方面，开发环境各自使用的数据库，可以隔离数据，避免干扰。 

### 默认数据
应用启动时，可以需要加载一些seed数据，hibernate支持通过import.sql导入seed数据。 

默认import.sql定义在classpath根目录，同时不同的数据库也要加载不同的import.sql，可以通过`jpa.properties.hiernate.hbm2ddl.import_files`定义import.sql的位置

DB Migrate
----------
数据库的移形可以使用flyway作为高层的工具对数据库模型进行修改、演变。由于数据库环境有多个，这样的话，开发环境和生产环境应使用不同的移形脚本。可以各profile的propertyes文件中定义`flyway.locations`的位置。

application-dev.properties

```
flyway.locations=classpath:db/migration/development
```

application-prod.properties

```
flyway.locations=classpath:db/migration/production
```

服务器
------
不依赖外部的服务器，便可减少开发环境的配置成本，提交效率。 Spring Boot本身支持将应用打包成Jar包，通过使用一个内嵌的HTTP服务器来运行应用。

`spring-boot-starter-web`默认包含了Tomcat作为内嵌服务器，可以通过声明其他服务器的依赖使用其他的数据库。

打包成war包需要将tomcat声明成provided依赖

```
providedRuntime 'org.springframework.boot:spring-boot-starter-tomcat'
```

### Debug
因为Spring Boot应用使用了内嵌的HTTP服务器，因此Debug也变的比较容易，不再需要依赖任何特殊的IDE插件或者扩展。只需要以Debug方式运行Main类即可。


测试
----

### Data Fixture
进行持久层的集成测试时，通常需要数据库的数据处于一个特定的状态。通过@Sql注解可以在支持@Test方法前执行特定的Sql，初始化数据库的状态。

另外每个测试方法执行完后，应该回滚数据库的状态，以便下一个测试方法执行。

```
@TransactionConfiguration(defaultRollback = true)
@Transactional
```

### 集成测试

集成测试有时需要启动Web服务器进行测试，这里再次体现了内嵌服务器的好处。应用可以单独地运行集成测试而不用依赖外部环境。

我们需要在一个Gradle/Maven任务中，单独运行所有的集成测试方法，在执行这个任务之前需要先启动Web服务器。Spring Boot的Gradle插件可以直接启动应用，但是无法以daemon的方式运行。

这里需要插件`gradle-tomcat-plugin`的配合。它继承自`war`插件，能够将构建war包然后部署到插件内嵌的tomcat中，支持以daemon的方式运行tomcat。

```
buildscript {
    repositories {
        jcenter()
    }

    dependencies {
        classpath 'com.bmuschko:gradle-tomcat-plugin:2.2.2'
    }
}

apply plugin: 'com.bmuschko.tomcat-base'

repositories {
    mavenCentral()
}

dependencies {
    def tomcatVersion = '7.0.59'
    tomcat "org.apache.tomcat.embed:tomcat-embed-core:${tomcatVersion}",
           "org.apache.tomcat.embed:tomcat-embed-logging-juli:${tomcatVersion}",
           "org.apache.tomcat.embed:tomcat-embed-jasper:${tomcatVersion}"
}

```

插件使用juli作为日志工具，但是spring boot的logging模块用了诸如`jcl-ovejulir-slf4j`、`jul-to-slf4j`、`log4j-over-slf4j`等的一些桥接库来统一底层使用的日志工具。 可能由于类加载器的问题，logging-juli是由System加载器加载的、jul-to-slf4j是由webapp加载的，会报如下错误：

```
java.lang.ClassCircularityError: java/util/logging/LogRecord
```

临时的解决方法是不使用`jul-to-slf4j`

```
configurations {
    compile.exclude module: "jul-to-slf4j"
}
```

最后tomcat插件的配置如下

```
task integrationTomcatRun(type: com.bmuschko.gradle.tomcat.tasks.TomcatRun) {
    contextPath = '/'
    httpPort = tomcatHttpPort
    stopPort = tomcatStopPort
    stopKey = tomcatStopKey
    daemon = true
}

task integrationTomcatStop(type: com.bmuschko.gradle.tomcat.tasks.TomcatStop) {
    stopPort = tomcatStopPort
    stopKey = tomcatStopKey
}

task integrationTest(type: Test) {
    include '**/*IntegrationTests.*'
    dependsOn integrationTomcatRun
    finalizedBy integrationTomcatStop
}

test {
    exclude '**/*IntegrationTests.*'
}

check.dependsOn integrationTest
integrationTest.mustRunAfter test
```

integrationTest匹配所有以IntegrationTest结尾的测试类。

QA
--
Sonarqube包括非常全面的代码检查，命名，规约，代码耦合，代码重复，代码复杂度，潜在的缺陷，单元测试的成功率，测试覆盖率等等。

Sonarqube服务器的搭建不在讨论范围之内。Sonarqube是通过一些本地的runner来分析代码，当然本地runner的检查规则还是需要从服务器上获取。

Sonarqube支持gradle plugin的本地分析，安装`gradle-sonarqube-plugin`即可。

```
apply plugin: "org.sonarqube"
```

插件非常的智能，会自动的的通过一些gradle项目参数来生成sonarqube的项目参数，所以不需要创建sonar-project.properties等文件。

同时我们还需要一些功能能够帮助我们在提交代码前就能进行分析，最好能够配合IDE之前在开发时显示错误。Idea有一个官方的sonarqube插件即可满足我们的需要，只是可惜目前还是不支持gradle多模块项目。只能使用社区版本的sonarqube插件。

社区版本的sonarqube插件支持以下功能

1. 从服务器上下载之前分析的issues，显示结果
2. 通过本地脚本分析生成sonar-report.json，显示结果

由于gradle插件不能生成sonar-report.json报告，所以这个插件的本地脚本分析相当不友好，只能另外配置sonar runner来执行本地分析。


### 测试覆盖
Sonarqube默认支持Jacoco生成的测试覆盖报告。

只需要应用插件

```
apply plugin: "jacoco"
```

默认执行test即会在/build/jacoco生成一个覆盖结果文件test.exec


持续构建
-------
CI工具使用Jenkins或许更为合适，拥有良好的插件机制可以更好的配合其他的功能、例如调用Sonarqube，发布文档，报告等等。

持续构建通过SCM远程出发，有代码提交即出发集成。以gitlab为例，首先在Jenkins中配置身份验证令牌，然后只需将Jenkins生成的触发URL添加gitlab的Web hooks

持续构建的过程应该包括如下

1. 从源码管理迁出最新的源代码
2. 调用`./gradlew clean build`进行构建（需要gradle plugin）
3. 调用`./gradlew sonarqube`进行代码检查（需要gradle plugin）
4. 远程部署应用
5. 发布构建成果，以gradle为例，应该是`**/build/libs/*.*`
6. 发布Junit测试报告，以gradle为例，应该是`**/build/test-results/*.xml`
7. 发布其他的HTML报告（需要html publish plugin）

### 自动部署
自动部署是要将生成war包部署到远程服务器上，立即看到构建的结果。可以通过`gradle-cargo-plugin`来实现。

配置如下

```
cargo {
    containerId = 'tomcat8x'
    port = 9000

    deployable {
        context = 'myawesomewebapp'
        file('/home/foo/bar/myawesomewebapp.war')
    }

    remote {
        hostname = 'cloud.internal.it'
        username = 'superuser'
        password = 'secretpwd'
    }

}
```


[spring-initializr]: https://start.spring.io/