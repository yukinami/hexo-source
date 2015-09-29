title: "Error: duplicate files during packaging of APK"
date: 2015-02-04 21:37:35
tags:
- Android
- 经验错误
---

错误
====
Android项目构建时报如下错

```
Error: duplicate files during packaging of APK /Users/snowblink/workspace/NoodlesMobile/app-stu/build/outputs/apk/xxx-debug-unaligned.apk
```

<!--more-->

原因
====
打包apk文件时，资源文件冲突。

解决
====
Maven
-----
```
<plugin>
	<groupId>com.jayway.maven.plugins.android.generation2</groupId>
	<artifactId>android-maven-plugin</artifactId>
	<extensions>true</extensions>
	<configuration>
		<extractDuplicates>true</extractDuplicates>
	</configuration>
</plugin>
```

Gradle
------
```
packagingOptions {
    exclude 'META-INF/notice.txt'
}
```

总结
====
[Android官方文档][1]对构建过程中打包apk的描述
>All non-compiled resources (such as images), compiled resources, and the .dex files are sent to the apkbuilder tool to be packaged into an .apk file.

打包apk的过程中构建工具会把所有资源文件拷贝到apk文件中，文件冲突即会报错

[1]: http://developer.android.com/sdk/installing/studio-build.html