title: Failed to complete gradle execution Cause
date: 2015-02-04 20:32:08
tags:
- Android
- 经验错误
---

错误
====
Android Studio 构建时报错 `failed to complete gradle execution Cause`
<!--more-->

使用Gradle命令查看具体原因，`./gradlew assembleDebug`

```
UNEXPECTED TOP-LEVEL EXCEPTION:
  	com.android.dx.cf.iface.ParseException: bad class file magic (cafebabe) or version (0034.0000)
```

原因
====
错误的原因是dex tool转换.class文件到.dex文件时，遇到了不兼容的Java Bytecode版本。
发生该错误的实际Android模块依赖了一个普通Java模块，使用的JDK8，编译时又没有指定source和target，所以默认生成了1.8的Java Bytecode。

解决
====
只需指定Java模块的编译字节码版本即可。

```
sourceCompatibility = 1.7
targetCompatibility = 1.7
```

总结
====
Android 4.4.2 (API 19) 以及以上只支持1.7的Java Bytecode
Android 4.4.2 (API 19）以下只支持1.6的Java Bytecode
`BuildToolsVersion 19`才开始支持Java 7