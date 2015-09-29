title: ScrollView
date: 2015-02-17 14:44:05
tags:
- Android
---

ScrollView 用来包裹一个视图，通过滚动页面使它能够超出物理的显示。ScrollView 本身是一个 FrameLayout, 以为着它只能包含一个子视图。子视图通常使用垂直方向的LinearLayout。

ScrollView 绝不应该和ListView一起使用，应为ListView会自己处理垂直方向的滚动。TextView也会处理自己的滚动，所以通常也不需要ScrollView，但是一起使用可以达到TextView在一个大容器中的效果

默认情况下，子视图的的layout_height的wrap_content，对子视图设定任何的高度都是无效的。如果fillViewport设置为true，那么子视图会填满整个ScollView。这在一些页面的按钮需要在最下方显示的时候是很有用的。