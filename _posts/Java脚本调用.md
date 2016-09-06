title: Java脚本调用
date: 2016-09-02 17:40:22
tags:
- Java
---

## JSR223

Java中调用其他脚本语言可以通过JSR223来实现。JSR223规范定义了脚本调用的抽象，只要拥有对应脚本的JSR223的实现，即可实现Java对对应脚本的调用。

例如，下面是对JS脚本的调用

```
import javax.script.*;
public class HelloWorld {
    public static void main(String[] args) throws ScriptException {
        ScriptEngineManager manager = new ScriptEngineManager();
        ScriptEngine engine = manager.getEngineByName("JavaScript");
        engine.eval("print ('Hello World')");
    }
}
```

<!-- more -->

### Ruby的调用

[JRuby]提供了Ruby脚本的的JSR223的实现。只需直接依赖JRuby的包

```
<dependency>
    <groupId>org.jruby</groupId>
    <artifactId>jruby</artifactId>
    <version>${jruby.version}</version>
</dependency>
```

引擎名为jruby

```
 ScriptEngineManager manager = new ScriptEngineManager();
        ScriptEngine engine = manager.getEngineByName("jruby");
        Bindings bindings = new SimpleBindings();
        bindings.put("message", "global variable");
        String script =
                "puts $message";
        engine.eval(script, bindings);
```

#### 使用gem

Jruby增强了RubyGems，它会寻找classpath下的specifications目录并自动地添加到`Gem.path`目录下，这意味着可以把整个gem repository打包成jar文件。

首先，需要通过安装需要的gem的来创建gem repository

```
$ java -jar jruby-complete-1.1.6.jar -S gem install -i ./chronic-gems chronic --no-rdoc --no-ri
Successfully installed rubyforge-1.0.2
Successfully installed rake-0.8.3
Successfully installed hoe-1.8.2
Successfully installed chronic-0.2.3
4 gems installed
```

然后打成jar包，注意jar包的名称不要和gem的名称一样（例如chronic.jar），否则当你`require 'chronic'`的时候JRuby会加载chronic.jar

```
$ jar cf chronic-gems.jar -C chronic-gems .
```

然后可以查看jar包中所包含的gem

```
java -jar jruby-complete-1.1.6.jar -S gem list
```

### NodeJS的调用

[J2V8]是V8引擎的Java Binding，它通过JNI调用来实现NodeJS的调用。下面是一个调用NPM模块的例子

```
public static void main(String[] args) {
  final NodeJS nodeJS = NodeJS.createNodeJS();
  final V8Object jimp = nodeJS.require(new File("path_to_jimp_module"));
 
  V8Function callback = new V8Function(nodeJS.getRuntime(), new JavaCallback() {	
    public Object invoke(V8Object receiver, V8Array parameters) {
      final V8Object image = parameters.getObject(1);
      executeJSFunction(image, "posterize", 7);
      executeJSFunction(image, "greyscale");
      executeJSFunction(image, "write",  "path_to_output");
      image.release();
      return null;
    }
  });
  executeJSFunction(jimp, "read", "path_to_image", callback);
 
  while(nodeJS.isRunning()) {
    nodeJS.handleMessage();
  }		
  callback.release();
  jimp.release();
  nodeJS.release();
}
```


## Spring Integration

Spring Integration 2.1开始支持对JSR223实现的集成调用。 通过Service Activator即可实现方法调用

```
<service-activator input-channel="input">
    <script:script lang="ruby" variables="foo=FOO, date-ref=dateBean">
        <script:variable name="bar" ref="barBean"/>
        <script:variable name="baz" value="bar"/>
        <![CDATA[
            payload.foo = foo
            payload.date = date
            payload.bar = bar
            payload.baz = baz
            payload
        ]]>
    </script:script>
</service-activator>
```

再配合Gateway，实现无缝的方法调用

```
<gateway service-interface="com.some.Foo"
             default-request-channel="input"
             default-reply-timeout="10000"
             default-reply-channel="reply" />
```


[JRuby]: http://jruby.org/
[J2V8]: https://github.com/eclipsesource/J2V8