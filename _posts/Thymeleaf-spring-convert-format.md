title: Thymeleaf spring convert/format
date: 2016-02-14 17:35:39
tags:
- Thymeleaf
---

Thymeleaf的双大括号语法，会调用Conversion Service来进行任何对象到String的转换。

- For variable expressions: ${{...}}
- For selection expressions: *{{...}}

这对大多数的属性标签都是有用的，但是th:field是个例外。