title: Spring Framework 4.1  – 处理静态web资源
tags:
  - Spring Framework
date: 2014-11-04 21:00:59
---

Spring Framework 4.1 的release已经有一阵子了，今天终于有点时间，看了下新特性中对于静态资源的灵活处理和转换，同时基于这种处理也提供了一种体验更好的开发方式，让我觉得非常兴奋。 先来看下对于这个新特性中的核心功能， [ResourceResolvers](http://docs.spring.io/spring-framework/docs/4.1.0.RC1/javadoc-api/org/springframework/web/servlet/resource/ResourceResolver.html) 和 [ResourceTransformers](http://docs.spring.io/spring-framework/docs/4.1.0.RC1/javadoc-api/org/springframework/web/servlet/resource/ResourceTransformer.html) 。

1.  ResourceResolvers用来将内部的URL解析为用户能够访问的外部公共的URL（ResourceResolver接口定义的是双向的解析）。
2.  ResourceTransformers用来修改资源的内容。

这两个功能相结合，不禁让我想起了Rails的Sprockets。Sprockets主要进行三项工作，资源文件的合并，压缩，以及对高级语言的预编译（coffeescript，sass）。那么Spring的这个特性是另外一个pipeline吗？ 官方博客给出的解释是这样的，

> 在Spring Framework 4.1中，我们使用依赖于优化的路径，这种优化在构件时使用最好的外部工具，在运行时使用Resolvers 和和Transformers 。

<!--more-->

比如说，我们可以用yeoman，grunt，bower搭建一个面向前端开发者的项目，在开发阶段，利用ResourceResolvers直接引用工程源代码的资源路径。在生产环境，再引用经过其他工具优化过的构建完成的资源。正式通过这种方式带给了我们一种全新的开发体验。 **Spring为这两个类提供了一些默认的实现类。**

<table>
<thead>
<tr>
<th>类名</th>
<th>目标</th>
</tr>
</thead>
<tbody>
<tr>
<td>[PathResourceResolver](http://docs.spring.io/spring-framework/docs/4.1.0.RC1/javadoc-api/org/springframework/web/servlet/resource/PathResourceResolver.html)</td>
<td>在配置的路径下查找匹配请求路径的资源</td>
</tr>
<tr>
<td>[CachingResourceResolver](http://docs.spring.io/spring-framework/docs/4.1.0.RC1/javadoc-api/org/springframework/web/servlet/resource/CachingResourceResolver.html)</td>
<td>从缓存实例中解析资源并且代理给下一个解析器</td>
</tr>
<tr>
<td>[GzipResourceResolver](http://docs.spring.io/spring-framework/docs/4.1.0.RC1/javadoc-api/org/springframework/web/servlet/resource/GzipResourceResolver.html)</td>
<td>当客户端支持gzip压缩时，查找资源的.gz扩展名形式的资源</td>
</tr>
<tr>
<td>[VersionResourceResolver](http://docs.spring.io/spring-framework/docs/4.1.0.RC1/javadoc-api/org/springframework/web/servlet/resource/VersionResourceResolver.html)</td>
<td>解析包含版本字符串的请求, 这个解析器通过改变资源的URL来设置HTTP缓存策略时很有用</td>
</tr>
</tbody>
</table>

<table>
<thead>
<tr>
<th>类名</th>
<th>目标</th>
</tr>
</thead>
<tbody>
<tr>
<td>[CssLinkResourceTransformer](http://docs.spring.io/spring-framework/docs/4.1.0.RC1/javadoc-api/org/springframework/web/servlet/resource/CssLinkResourceTransformer.html)</td>
<td>修改CSS文件中得链接来匹配应该暴露给客户端的公共URL</td>
</tr>
<tr>
<td>[CachingResourceTransformer](http://docs.spring.io/spring-framework/docs/4.1.0.RC1/javadoc-api/org/springframework/web/servlet/resource/CachingResourceTransformer.html)</td>
<td>缓存transformations的结果或者代理给下一个Transformer</td>
</tr>
<tr>
<td>[AppCacheManifestTransfomer](http://docs.spring.io/spring-framework/docs/4.1.0.RC1/javadoc-api/org/springframework/web/servlet/resource/AppCacheManifestTransfomer.html)</td>
<td>帮助处理HTML5离线应用的AppCache清单内的文件</td>
</tr>
</tbody>
</table>


**最后来看点例子**。 首先是目录结构， [Spring Resource Handling showcase application](https://github.com/bclozel/spring-resource-handling)：
<pre class="crayon-plain-tag">spring-app/
|- build.gradle
|- client/
|  |- src/
|  |  |- css/
|  |  |- js/
|  |     |- main.js
|  |- test/
|  |- build.gradle
|  |- gulpfile.js
|- server/
|  |- src/main/java/
|  |&ndash; build.gradle</pre>

这样的项目结构有两个好处，

1.  <span style="font-size: 12px;">更好的开发体验，因为在开发阶段引用的是源代码。</span>
2.  <span style="font-size: 12px;">生产环境中优化的性能，静态资源通过构建得以优化并且打包为webJar。</span>

**配置实例**

1\. 注册Resolver和Transformer
<pre class="crayon-plain-tag">AppCacheManifestTransformer appCacheTransformer = new AppCacheManifestTransformer();
VersionResourceResolver versionResolver = new VersionResourceResolver()
.addFixedVersionStrategy(version, &quot;/**/*.js&quot;)
.addContentVersionStrategy(&quot;/**&quot;);

registry.addResourceHandler(&quot;/**&quot;)
.addResourceLocations(location)
.setCachePeriod(cachePeriod)
.resourceChain(useResourceCache)
.addResolver(versionResolver)
.addTransformer(appCacheTransformer);</pre>

2\. 手动解析
<pre class="crayon-plain-tag">ResourceUrlProvider.getForLookupPath()</pre>

3\. 注册

ResourceUrlEncodingFilter，调用Response的encodeURL进行解析
<pre class="crayon-plain-tag">@Bean
public ResourceUrlEncodingFilter resourceUrlEncodingFilter() {
ResourceUrlEncodingFilter filter = new ResourceUrlEncodingFilter();

return filter;
}</pre>

ResourceUrlEncodingFilter是一个包装HttpServletResponse的过滤器，它覆盖了{HttpServletResponse#encodeURL(String) encodeUR} 方法，将内部URL转换为外部的公共URL。其内部也是使用了ResourceUrlProvider来进行转换的。

ResourceUrlProvider是获取外部URL路径的转换的核心组件，其内部定了Map&lt;String, ResourceHttpRequestHandler&gt; handlerMap用来进行链式的解析。

ResourceHttpRequestHandler是&lt;mvc:resource /&gt; 标签的具体实现，4.1版本ResourceHttpRequestHandler内部使用ResourceResolver以及ResourceTransformer来进行解析处理。