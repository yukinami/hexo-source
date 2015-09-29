title: 为什么AMD
tags:
  - 前端开发
date: 2014-11-05 17:48:54
---

关于JS的模块化，阮一峰的博客上有篇很好的文章，请参考，[模块的写法](http://www.ruanyifeng.com/blog/2012/10/javascript_module.html)，[AMD规范](http://www.ruanyifeng.com/blog/2012/10/asynchronous_module_definition.html)。

本文翻译自RequireJS官方文档 [WHY AMD?](http://requirejs.org/docs/whyamd.html), 更为详细地阐述了为什么需要AMD。

### 模块化的目的

什么是JavaScript模块？它们的目的是什么？

*   定义：如何将一段代码封装为一个有用的单元，并且怎么描述它的功能、怎样为这个模块导出一个值
*   依赖参考：如何引用其他代码单元

### 当今的WEB

<pre class="crayon-plain-tag">(function () {
    var $ = this.jQuery;

    this.myExample = function () {};
}());</pre>

<!--more-->

当今的JavasScript代码片段是如何定义的？

*   通过立即执行函数定义
*   通过HTML脚本标签加载的全局变量名来引用依赖
*   依赖的描述非常脆弱：开发者需要知道正确的依赖顺序。比如说包含Backbone的文件不能出现在jQuery标签前面。
*   它需要外部的工具来替换一系列的脚本标签为一个部署优化过的文件。

这在大工程的管理上是非常困难的，特别是当脚本有很多重叠嵌套的依赖的时候。手写的Script标签扩展性差，并且它失去了根据需求加载脚本的能力。

### COMMONJS

<pre class="crayon-plain-tag">var $ = require('jquery');
exports.myExample = function () {};</pre>

最初的 [CommonJS (CJS) list](http://groups.google.com/group/commonjs) 参与者决定为如今的JavaScript语言制定一种模块格式，但是没有必要局限于浏览器JS环境。希望在浏览器中使用一些权宜之计并且希望影响浏览器开发商开发一些能够使他们的模块格式在本地工作得更好的方案。

这些权宜之计是：

*   使用服务器将CJM模块翻译成能够在浏览器中使用的东西
*   使用XMLHttpRequest (XHR) 来加载模块的文本内容，然后在浏览器中做一些转换

CJS 模块只允许一个文件一个模块，所以需要一个传输格式来处理为了优化目的多个模块绑定到一个文件的情况。

通过这种方式，CommonJS小组能够解决依赖引用，以及如何处理循环依赖，如何取得关于当前模块的一些属性。然而，他们没有完全接受一些在浏览器环境中无法被改变但是仍然影响模块设计的东西：

*   网络加载
*   固有的异步

这意味着他们更多地把压力负担给web开发者来实现这个格式，并且上述权宜之计意味着更难的调试。基于eval调试或者调试一个由多个文件合并的文件有实用上的弱点。这些弱点可能会某一天在浏览器工具中得到解决，但是最终的结果：在普通的JS环境中使用CommonJS模块，在今天的浏览器中，不是最佳的。

### AMD

<pre class="crayon-plain-tag">define(['jquery'] , function ($) {
    return function () {};
});</pre>

AMD格式来源于对一种比当今的“写一堆有隐式依赖的并且必须手动控制的Script标签“更好的，并且在浏览器中容易直接使用的需求。它拥有更好的debug特性，并且不需要特定于服务器端的工具的。它的前身是Dojo的现实实际中的经验——使用XHR+eval并且想要避免它的弱点。

它是对当前web的全局script标签的改进，体现在：

*   使用CommonJS的字符串IDs作为依赖的实践。清晰的依赖声明并且避免使用全局的东西。
*   IDs能够被映射为不同的路径。这允许替换具体的实现。这对于创建单元测试的mock对象是相当有利的。对于上面的样例代码，它只是期望实现了jQuery API和行为的东西，不是非得是jQuery。
*   封装了模块的定义。给你一个避免污染全局命名空间的工具。
*   定义模块值的清晰路径。要么使用“return value；”，或者使用CommonJS的对循环依赖非常有用的“export”的惯用语法。

它是对CommonJS的模块的改进，体现在：

*   它在浏览器中工作得更好，它有最少的陷阱。其他的方式有debug的问题，跨域/CDS使用的问题，file://的使用的问题，需要服务器工具的问题。
*   定义了一种在一个文件中包含多个模块的方式。用CommonJS的术语来说，是“传输格式”，并且那个小组没有在传输格式上达成一致。
*   允许将函数作为返回值。这对于构造函数非常有用。这点在CommonJS中更为笨拙，CommonJS中必须在输出的对象上定义一个属性。Node支持module.exports = function () {}，但是这不是CommonJS的规范。

### 模块定义

使用JavaScript的函数来封装模块在[module pattern](http://www.adequatelygood.com/2010/3/JavaScript-Module-Pattern-In-Depth)有写到：
<pre class="crayon-plain-tag">(function () {
   this.myGlobal = function () {};
}());</pre>

这种模块定义依赖于在全局对象是附加属性来导出模块值，并且在这个模型中很难声明依赖。依赖被认为是在函数执行是即刻可用的。这限制了对依赖的加载策略。

AMD解决这些问题，通过：

*   通过调用define()方法注册工厂方法，而不是立刻执行
*   通过string数组传递依赖，而不是抓取全局的
*   当所有的依赖加载并执行完后，工厂方法只会执行一次
*   将依赖模块做为参数传给工厂方法
<pre class="crayon-plain-tag">//Calling define with a dependency array and a factory function
define(['dep1', 'dep2'], function (dep1, dep2) {

    //Define the module value by returning a value.
    return function () {};
});</pre>