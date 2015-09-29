title: FLAG_ACTIVITY_CLEAR_TOP launches new instance
date: 2015-07-01 09:40:53
tags:
- Android
- 经验错误
---

错误
====
在使用`NavUtils.navigateUpFromSameTask(this)`导航到父Activity时，页面切换的动画是从右往左滑动，这样的动画看起来不像是返回动画，而是打开了一个新的页面的动画。

<!--more-->

原因
====
SDK版本小于16时，`NavUtilsImpl`的实现是`NavUtilsImplBase`，其中的`navigateUpTo`方法是实现是添加了flag`Intent.FLAG_ACTIVITY_CLEAR_TOP`。

关于`Intent.FLAG_ACTIVITY_CLEAR_TOP`，官方文档中是这样描述的

> If the activity being started is already running in the current task, then instead of launching a new
> instance of that activity, all of the other activities on top of it are destroyed and this intent is 
> delivered to the resumed instance of the activity (now on top), through onNewIntent()).

但是实际情况，单单一个`Intent.FLAG_ACTIVITY_CLEAR_TOP`还是会创建新的实例，在stackoverflow有一个相关的[问题][1]。

需要`Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_SINGLE_TOP`才会执行`onNewIntent`。


[1]: http://stackoverflow.com/questions/4342761/how-do-you-use-intent-flag-activity-clear-top-to-clear-the-activity-stack