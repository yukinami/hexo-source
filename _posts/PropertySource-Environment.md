title: 'PropertySource & Environment'
date: 2016-02-14 16:21:58
tags:
- Spring Framework
---

在Spring 3.1之前，注册一个配置文件的唯一方法就是

```
<context:property-placeholder location="some.properties"/>
```

Spring 3.1引入了`PropertySource`和`Environment`接口，可以通过@PropertySource注解来注册

```
@Configuration
@PropertySource("classpath:some.properties")
public class ApplicationConfiguration
```

并且从Spring 3.1之后，`<context:property-placeholder>`实际底层注册的是`PropertySourcesPlaceholderConfigurer`对象。 这个对象具有更好的扩展性并且和`Environment`，`PropertySource`接口交互。

<!--more-->

## 使用PropertySource

推荐的使用方式是

```
@Configuration
@PropertySource("classpath:some.properties")
public class ApplicationConfiguration
```

然后注入environment

```
@Inject
private Environment environment;
```

通过`Environment`实例来读取配置

```
environment.getProperty("foo.bar")
````

### 在Spring 3.2中使用@Value注解

使用@PropertySource注解并没有注册`PropertySourcesPlaceholderConfigurer`(或者`PropertyPlaceholderConfigurer`)对象。@Value和Environment的机制是独立的，并且environment是推荐的方式。

如果仍想使用@Value注解，可以注册`PropertySourcesPlaceholderConfigurer`对象来让其调用`Environment`接口达到目的。

## Mocking PropertySource

Mock一个PropertySource，添加如下代码到测试代码中

```
public static class PropertyMockingApplicationContextInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        MutablePropertySources propertySources = applicationContext.getEnvironment().getPropertySources();
        MockPropertySource mockEnvVars = new MockPropertySource().withProperty("bundling.enabled", false);
        propertySources.replace(StandardEnvironment.SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, mockEnvVars);
    }
}
```

```
@ContextConfiguration(classes = ApplicationConfiguration.class, initializers = MyIntegrationTest.TestApplicationContextInitializer.class)
```

或者直接添加Environment的mock对象
```
public static class PropertyMockingApplicationContextInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        MockEnvironment mockEnvironment = new MockEnvironment();
        applicationContext.setEnvironment(mockEnvironment);
    }
}
```

[UsingPropertySourceAndEnvironment]: http://blog.jamesdbloom.com/UsingPropertySourceAndEnvironment.html

