title: JVM热部署
date: 2015-07-14 16:30:56
tags:
- Java
- 热部署
---

JVM Hot Swapping
================
从Java1.4启，JVM引入了HotSwap，能够在Debug的时候更新类的字节码，所有新式的IDE（包括Eclipse、IDEA和NetBeans）都支持这一技术。但是它只能用来更新方法体，这一定程度上限制了它的实用性。

<!--more-->

在Idea中的使用
------------

1. HotSwap要求必须以Debug方法启动JVM
2. 修改文件
3. 菜单选择 Run | Reload Changed Classes 重新加载所有修改的类，或者 Build | Compile "class_name" 重新编译加载当前的类


JRebel
======
JRebel是另外一种更新类的技术。他通过监控磁盘上已经编译的.class文件来实现类更新，所以它不依赖于IDE。同时对于更新类的字节码也没有HotSwap那么多的限制


关于HotSwap和JRebel这里有更详细的[介绍][1]

DCEVM 
======
[DCEVM][2]的全称是Dynamic Code Evolution VM，它是实现在Oracle’s Java HotSpot VM之上，对其进行了扩展、修改，所以它对字节码的更新也没有什么限制。可以说是相对JRebel的一个不错的免费的选择。

Github上有一个它的fork[版本][3]，针对OpenJDK进行了一些加强。

Spring-Loaded
==============
Spring-Loaded是用来重新加载类文件的JVM agent。不像HotSwap，Spring-Loaded支持方法、属性以及构造方法的增删改，同时也支持注解的增删改。

ming使用
----
如下方式启动JVM

```
java -javaagent:<pathTo>/springloaded-{VERSION}.jar -noverify SomeJavaClass
```

之后对类文件的修改即可更新加载到JVM中的类。

在IDE中的使用
------------
Spring-Loaded并不需要以Debug方式启动JVM，直接修改文件，菜单选择 Run | Reload Changed Classes 重新加载所有修改的类，或者 Build | Compile "class_name" 重新编译加载当前的类即可



[1]: http://zeroturnaround.com/blog/reloading_java_classes_401_hotswap_jrebel/
[2]: http://ssw.jku.at/dcevm/
[3]: https://github.com/dcevm/dcevm