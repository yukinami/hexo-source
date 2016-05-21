title: Hibernate one-to-one 懒加载的问题
date: 2016-05-21 14:24:29
tags:
- Hibernate
---

## 问题

Hibernate在使用OneToOne的反向关联或者使用主键关联（PrimaryKeyJoinColumn）时，会出来懒加载不起的作用的情况。究其原因，Hibernate的能够进行懒加载的前提是返回的关联对象是个代理对象。如果Hibernate不能确定关联的对象是否为空，那么他们不能直接返回代理对象，因为代理对象本身就是不为空的，它不得不去检查关联对象是否存在。但是用SELECT语句去检查存在性，还不如索性直接把查询结果返回，这也就导致了懒加载的失效。

<!-- more -->

上面的两种情况都是无法确定关联对象是否存在的

- 反向关联  由于关联的外键并不存在于当前表，而是在关联表，必须把当前记录的主键作为条件去关联表的外键中去匹配才能确定关联对象是否存在
- 主键关联  由于关联的对象表的主键是使用的当前表的主键，单单查看当前表也无法确定关联对象的存在性

所以也就导致这两种情况下的关联属性的懒加载是无法生效的。

## 解决

## 强制存在

如果在映射文件中就明确定义出来关联对象不可能为空，那么Hibernate就没有必要去检查它的存在性，直接返回代理对象即可。 当前前提是业务的角度来说关联对象确实不可能为空。

## 使用OneToMany来替代OneToOne

使用假的OneToMany来替代OneToOne，实际使用的时候始终取集合的第一个元素。因为Many端永远不会返回空，最多返回空集合。但是这个方式会对JPQL/HQL产生一定的影响

## 使用字节码织入

不使用代理对象，而是使用字节码织入，最终调用关联对象的任何方法时才会触发懒加载。对于字节码的织入有编译时织入和运行时织入两种方式。编译时织入可以通过Hibernate提供的ant工具来实现。运行时织入则依赖的JavaEE环境或者使用Spring的运行时织入技术。

无论哪种情况，最终都需要使用@LazyToOne(LazyToOneOption.NO_PROXY) 注解来显示的告诉框架不适用代理对象，以及把Hibernate的`hibernate.ejb.use_class_enhancer`参数设置为true。

### 启用Spring's LTW

Spring的LTW的核心组件是`LoadTimeWeaver`接口，`LoadTimeWeaver`的职责是负责在运行时添加一个和多个`java.lang.instrument.ClassFileTransformers`到`ClassLoader`中。

要启用Spring的LTW支持，首先需要配置`LoadTimeWeaver`

```
@Configuration
@EnableLoadTimeWeaving
public class AppConfig {

}
````

上述的配置会定义并注册一系列的LTW相关的Bean。默认的`LoadTimeWeaver`的实现类`DefaultContextLoadTimeWeaver`，它是一个包装类，他会装饰自动检测到的`LoadTimeWeaver`的实现类。


- 对于WebLogic, WebSphere, Resin, GlassFish, JBoss这些服务器的最近的一些版本提供的类加载器是支持本地织入的，所以只需要直接激活LTW即可。
- 对于Tomcat6，默认的类加载器不支持`class transformation`，需要使用Spring提供的类加载器`TomcatInstrumentableClassLoader`
    ```
    <Context path="/myWebApp" docBase="/my/webApp/location">
    <Loader
        loaderClass="org.springframework.instrument.classloading.tomcat.TomcatInstrumentableClassLoader"/>
    </Context>
    ```
- 对于普通的Java应用，JDK agent是唯一的解决办法。Spring提供了`InstrumentationLoadTimeWeaver`，但是需要Spring特定的VM agent。
    ```
    -javaagent:/path/to/org.springframework.instrument-{version}.jar
    ```

### 给EntityManagerFactory设置LoadTimeWeaver

Spring 4.3之前可能需要手动的设置LoadTimeWeaver，[这里][4]是相关的问题。手动设置的方法是调用`LocalContainerEntityManagerFactoryBean#setLoadTimeWeaver`方法。对于Spring Boot，可以用下面的方式来设置

```
@Bean
    @ConditionalOnMissingBean
    public EntityManagerFactoryBuilder entityManagerFactoryBuilder(
            JpaVendorAdapter jpaVendorAdapter) {
        EntityManagerFactoryBuilder builder = new EntityManagerFactoryBuilder(
                jpaVendorAdapter, this.jpaProperties.getProperties(),
                this.persistenceUnitManager);
        builder.setCallback(getVendorCallback());
        return builder;
    }

    protected EntityManagerFactoryBuilder.EntityManagerFactoryBeanCallback getVendorCallback() {
        return factory -> factory.setLoadTimeWeaver(loadTimeWeaver);
    }
```    


[1]: https://developer.jboss.org/wiki/SomeExplanationsOnLazyLoadingone-to-one
[2]: http://justonjava.blogspot.sg/2010/09/lazy-one-to-one-and-one-to-many.html
[3]: http://stackoverflow.com/questions/18423019/how-to-enable-load-time-runtime-weaving-with-hibernate-jpa-and-spring-framewor
[4]: https://jira.spring.io/browse/SPR-10856