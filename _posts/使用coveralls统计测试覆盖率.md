title: 使用coveralls统计测试覆盖率
date: 2016-05-27 21:14:07
tags:
- CI
- 测试
---

[coveralls]可以持续跟踪代码的测试覆盖率。 为了让它能够跟踪到测试覆盖率，需要测试的时候生成覆盖率数据，并提供给coveralls service。

<!-- more -->

## 生成测试覆盖率数据

jacoco和cobertura都可以用来统计代码的测试覆盖率，并且各自都有gradle和maven的插件。下面以gradle为例进行讨论。

```
apply plugin: "jacoco"
```

通过上述配置启用jacoco插件。启用后，按照官方文档说法，如果同时启用了Java插件的话，那么任务jacocoTestReport将会被创建并且它依赖于test任务，但是实际运行的时候发现它并没有依赖于test任务

```
:compileJava
:compileGroovy
:processResources
:classes
:jacocoTestReport SKIPPED
```

并且直接运行的话，它会被跳过。

jacocoTestReport任务是用来发布jacoco报告的，它依赖于jacoco的测试覆盖率数据文件`text.exec`。 如果启用了jacoco插件，测试覆盖率数据文件会在test任务执行的时候生成，位于`build/jacoco`目录下。

在生成这个文件的前提下，执行jacocoTestReport任务，就会在`build/reports/tests/jacoco`目录生成报告，默认生成HTML格式的报告。

## 上传coveralls

jacoco生成的报告需要上传给coveralls才能显示。Maven的话通过[coveralls-maven-plugin]可以实现，gradle则使用插件[coveralls-gradle-plugin]。

以[coveralls-gradle-plugin]为例

```
plugins {
    id 'jacoco'
    id 'com.github.kt3k.coveralls' version '2.6.3'
}
```

它插件依赖于jacoco生成的XML报告，因此需要配置jacoco

```
jacocoTestReport {
	reports {
		xml.enabled = true
		html.enabled = true
	}
}
```

## CI服务器

最后coveralls需要配合一个CI服务器，持续地对代码的测试覆盖率进行统计，支持[Travis CI]、Jenkins等多种CI服务器。

以Travis为例，需要在构建的最后执行上面的提到的，生成测试覆盖率报告并上传的任务

```
after_success:
- ./gradlew jacocoTestReport coveralls
````


[coveralls]: https://coveralls.io/
[coveralls-maven-plugin]: https://github.com/trautonen/coveralls-maven-plugin
[coveralls-gradle-plugin]: https://github.com/kt3k/coveralls-gradle-plugin
[Travis CI]: https://travis-ci.org/