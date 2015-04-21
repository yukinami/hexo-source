title: Junit Rules
date: 2015-04-21 10:41:28
tags:
- Java
- 测试
---

Interceptors
============
Junit4.7引入了一个新功能叫做"Interceptors"，旨在以一个更为清晰简单的API，带回JUnit`meta-testing`的能力。但是之后又被重新命名为"Rule"。

Meta-testing
------------
Junit擅长表达尽量少的测试逻辑。在Junit3中，你能够以不同的方式控制测试运行过程。Junit4简化的代价就是失去了这个“meta-testing”。这不影响简单测试，但是限制了更为强大的测试。现在Junit4重新把meta-testing带了回来。

比如说，你想输出一个log当每个测试失败的时候。

<!--more-->

```
@Interceptor
public StatementInterceptor logger=
new LoggingInterceptor();
```

声明了@Interceptor，logger会在测试运行之前被调用。当前，interceptors并不是准表的test runner的一部分，所以你必须用一个特殊的runner来运行测试。

```
@RunWith(Interceptors.class)
public class MyLoggingTest {
@Interceptor
public StatementInterceptor logger=
new LoggingInterceptor();

}
```



Rules
=====
[Rules][1]允许灵活的添加或者重新定义测试case中的每一个测试方法行为。测试者可以重复利用或者扩展Junit框架提供的Rules，或者书写自己的Rules。


TemporaryFolder Rule
--------------------
TemporaryFolder Rule允许创建文件和文件夹并且在测试方法完成之后删除它们

```
public static class HasTempFolder {
  @Rule
  public TemporaryFolder folder = new TemporaryFolder();

  @Test
  public void testUsingTempFolder() throws IOException {
    File createdFile = folder.newFile("myfile.txt");
    File createdFolder = folder.newFolder("subfolder");
    // ...
  }
} 
```

ExternalResource Rules
-----------------------
ExternalResource Rule是可以在测试之前设置一个外部资源 (a file, socket, server, database connection, etc.)，并且保证在结束时关闭它们的Rule的基类，例如TemporaryFolder Rule。


```
public static class UsesExternalResource {
  Server myServer = new Server();

  @Rule
  public ExternalResource resource = new ExternalResource() {
    @Override
    protected void before() throws Throwable {
      myServer.connect();
    };

    @Override
    protected void after() {
      myServer.disconnect();
    };
  };

  @Test
  public void testFoo() {
    new Client().run(myServer);
  }
}
```

ErrorCollector Rule
--------------------
ErrorCollector Rule允许测试继续执行当发生错误的时候（比如说，来收集表中的所有的错误行，然后汇总报告）

```
public static class UsesErrorCollectorTwice {
  @Rule
  public ErrorCollector collector= new ErrorCollector();

  @Test
  public void example() {
    collector.addError(new Throwable("first thing went wrong"));
    collector.addError(new Throwable("second thing went wrong"));
  }
}
```

Verifier Rule
-------------
Verifier Rule是ErrorCollector等Rules的基类，当verification check失败的时候会导致测试方法失败。

```
public static class ErrorLogVerifier {
    private ErrorLog errorLog = new ErrorLog();

    @Rule
    public Verifier verifier = new Verifier() {
       @Override public void verify() {
          assertTrue(errorLog.isEmpty());
       }
    }

    @Test public void testThatMightWriteErrorLog() {
       // ...
    }
 }

```



[1]: https://github.com/junit-team/junit/wiki/Rules
[2]: http://www.threeriversinstitute.org/blog/?p=155