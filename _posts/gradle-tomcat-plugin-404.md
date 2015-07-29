title: gradle-tomcat-plugin 404
date: 2015-07-29 15:30:50
tags:
- Gradle
- 经验错误
---

问题
====
执行gradle-tomcat-plugin的tomcatRun任务时，服务器正常启动，没有任何异常，输出的log也没有异常，但是访问时返回404页面

<!--more-->

原因
====
gradle-tomcat-plugin部署应用时，插件会去查找 src/main/webapp 目录。如果没有找到这个目录，则应用不会被部署。出现这个问题的时候，我的代码是从git服务器上拉下来的，确实发现服务器上没有webapp这个目录，但是本地是有这个目录的。 由于是空目录，git的无法跟踪空目录，也就导致这个目录没有被提交上去。

关于git跟踪空目录参考[这里][git-track-empty-directories]

解决
====
只要把webapp这个目录提交上去就可以解决了。 目录下添加.gitignore

```
# Ignore everything in this directory
*
# Except this file
!.gitignore
```

[git-track-empty-directories]: https://git.wiki.kernel.org/index.php/Git_FAQ#Can_I_add_empty_directories.3F