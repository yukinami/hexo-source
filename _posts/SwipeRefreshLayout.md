title: SwipeRefreshLayout
date: 2015-02-17 15:02:57
tags:
- Android
---

SwipeRefreshLayout 可以用于通过下滑的手势刷新View内容的情况。使用起来很简单，用SwipeRefreshLayout包裹需要刷新的View即可。

```
<android.support.v4.widget.SwipeRefreshLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/swipe_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

    <LinearLayout android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <ListView
            android:id="@android:id/list"
            style="@style/ListView"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
    </LinearLayout>

</android.support.v4.widget.SwipeRefreshLayout>
```
<!--more-->

初始化View的Activity应该注册一个`OnRefreshListener`来处理手势完成时需要执行的（刷新）动作。当手势结束的时候，Refreshing指示器会一直显示，如果监听者决定不刷新，或者已经刷新完了，那么要调用`setRefreshing(false)`来取消Refreshing指示器。如果Activity需要单独显示Refreshing指示器，也可以调用`setRefreshing(true)`。

这个Layout应该作为需要刷新内容的父视图，并且只支持一个子视图。这个子视图将成为手势的目标，并且会被强制匹配`SwipeRefreshLayout`的宽高。

`SwipeRefreshLayout`不提供可访问的事件；因此，应该提供一个菜单项来允许，当滑动手势时刷新内容。


不知道是不是TextView自己有处理滚动的原因或者其他，`SwipeRefreshLayout`单独和`TextView`使用时会很奇怪，往下滑动时Refreshing指示器不会正常显示，只有到松手时，才会突然出现一次又消失。解决这个问题可以通过在TextView外面在包括一个ScollView。

```
<android.support.v4.widget.SwipeRefreshLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/swipe_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

    <LinearLayout android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <ListView
            android:id="@android:id/list"
            style="@style/ListView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:visibility="gone" />

        <ScrollView
            android:layout_width="match_parent"
            android:layout_height="match_parent" >

            <TextView
                android:id="@android:id/empty"
                style="@style/ListSubtitleText"
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:gravity="center"
                android:layout_gravity="center"
                android:text="@string/default_empty" />
        </ScrollView>
    </LinearLayout>

</android.support.v4.widget.SwipeRefreshLayout>
```
