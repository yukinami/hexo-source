title: Fatal signal 11
date: 2015-03-16 13:23:02
tags:
- Android
- 经验错误
---

错误
====
多个`AsyncTask`、`Loader`同时执行的时候，偶尔会闪退。 错误信息大致如下：
```
@@@ ABORTING: INVALID HEAP ADDRESS IN dlfree
Fatal signal 11 (SIGSEGV) at 0xdeadbaad (code=1)
```

<!--more-->
原因
====
Stackoverflow上有一个类型的[问题][1]，但是它的原因是在多个线程中调用了`BluetoothSocket#close`方法。回到自己的问题上来，我这边发生的问题也极有可能在网络操作方法，而不是异步任务本身。由于我的项目使用了`spring android`来对远程API进行请求操作的，所以最终问题定位到`RestTempalte`上。起初怀疑是使用的网络模块的问题，`spring android`可以使用`apache httpclient`、`okhttp`等作为http请求模块，但是逐一尝试还是解决不了问题。那问题只能出在`RestTempalte`本身了。Spring论坛上找到一个类似的[问题][1]，解决的办法是在`RestTemplate`初始化的代码上加上同步块，但是更为根本的原因还是不太清楚。

解决
====
我的项目是使用Roboguice来注入各种service供异步任务调用的，各个service对象注入的时候都会初始化`RestTempalte`。但是`RestTempalte`本身是线程安全的，所以所有的service使用同一个`RestTempalte`应该是没问题的。最终的修改就是service不再初始化`RestTempalte`，还是依赖注入一个容器中单例的`RestTempalte`。



[1]: http://stackoverflow.com/questions/10662446/invalid-heap-address-and-fatal-signal-11
[2]: http://forum.spring.io/forum/spring-projects/android/113537-is-new-resttemplate-thread-safe-android