title: Spring AOP的实现
date: 2016-05-21 17:43:51
tags:
- Spring
- AOP
---

## 概念

- Aspect 切面，指的是切分多个类的模块化的关注点，包括Pointcut或Advice

  ```
  @Aspect
  public class NotVeryUsefulAspect {

  }
  ```
  
<!-- more -->
- Join point 程序的执行点， 可用于插入代码 （在Spring里面指的是方法的执行）
- Advice 对特定的Join point执行的动作
  ```
  @Before("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
    public void doAccessCheck() {
        // ...
    }
  ```
- Pointcut 匹配Join point的断言

  ```
  @Pointcut("execution(* transfer(..))")// the pointcut expression
  private void anyOldTransfer() {}// the pointcut signature
  ```

- Introduction 定义额外的方法或字段
- Target Object 被一个或多个切面通知的目标对象
- AOP proxy 由AOP框架创建的代理对象
- Weaving 连接切面和应用对象来实现切面的功能

AOP的实现关键在于AOP代理的创建。代理对象可以分为静态代理和动态代理。

- 静态代理的实现  通过织入来创建静态的代理对象，AspectJ的实现就是这种方式
- 动态代理的实现  通过JDK动态代理或者CGlib来创建动态的代理对象，Spring AOP的实现就是这种方式


## Spring AOP

Spring AOP采用的是动态代理的实现。配置AOP方式可以采用AspectJ的注解或者基于Spring的Schema的XML配置。


### 基于AspectJ的AOP配置

```
@Aspect
public class ConcurrentOperationExecutor implements Ordered {

    private static final int DEFAULT_MAX_RETRIES = 2;

    private int maxRetries = DEFAULT_MAX_RETRIES;
    private int order = 1;

    public void setMaxRetries(int maxRetries) {
        this.maxRetries = maxRetries;
    }

    public int getOrder() {
        return this.order;
    }

    public void setOrder(int order) {
        this.order = order;
    }

    @Around("com.xyz.myapp.SystemArchitecture.businessService()")
    public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
        int numAttempts = 0;
        PessimisticLockingFailureException lockFailureException;
        do {
            numAttempts++;
            try {
                return pjp.proceed();
            }
            catch(PessimisticLockingFailureException ex) {
                lockFailureException = ex;
            }
        } while(numAttempts <= this.maxRetries);
        throw lockFailureException;
    }

}
```

```
<aop:aspectj-autoproxy/>

<bean id="concurrentOperationExecutor" class="com.xyz.myapp.service.impl.ConcurrentOperationExecutor">
    <property name="maxRetries" value="3"/>
    <property name="order" value="100"/>
</bean>
```

### 基于Schema的AOP配置

```
<aop:config>

    <aop:aspect id="concurrentOperationRetry" ref="concurrentOperationExecutor">

        <aop:pointcut id="idempotentOperation"
            expression="execution(* com.xyz.myapp.service.*.*(..))"/>

        <aop:around
            pointcut-ref="idempotentOperation"
            method="doConcurrentOperation"/>

    </aop:aspect>

</aop:config>

<bean id="concurrentOperationExecutor"
    class="com.xyz.myapp.service.impl.ConcurrentOperationExecutor">
        <property name="maxRetries" value="3"/>
        <property name="order" value="100"/>
</bean>
```

### Spring中使用AspectJ

AspectJ的通过静态代理来实现AOP的织入的。根据织入时机的不同，又分为编译时织入和类加载时织入（LTW）。Spring对AspjectJ的LTW进行了一定的增强，使得LTW能够针对于单个ClassLoader而不是整个JVM。

下面是个例子

首先定义AOP

```
package foo;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.util.StopWatch;
import org.springframework.core.annotation.Order;

@Aspect
public class ProfilingAspect {

    @Around("methodsToBeProfiled()")
    public Object profile(ProceedingJoinPoint pjp) throws Throwable {
        StopWatch sw = new StopWatch(getClass().getSimpleName());
        try {
            sw.start(pjp.getSignature().getName());
            return pjp.proceed();
        } finally {
            sw.stop();
            System.out.println(sw.prettyPrint());
        }
    }

    @Pointcut("execution(public * foo..*.*(..))")
    public void methodsToBeProfiled(){}
}
```

然后定义需要织入的对象

```
<!DOCTYPE aspectj PUBLIC "-//AspectJ//DTD//EN" "http://www.eclipse.org/aspectj/dtd/aspectj.dtd">
<aspectj>

    <weaver>
        <!-- only weave classes in our application-specific packages -->
        <include within="foo.*"/>
    </weaver>

    <aspects>
        <!-- weave in just this aspect -->
        <aspect name="foo.ProfilingAspect"/>
    </aspects>

</aspectj>
```

最后启用Spring的LTW

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- a service object; we will be profiling its methods -->
    <bean id="entitlementCalculationService"
            class="foo.StubEntitlementCalculationService"/>

    <!-- this switches on the load-time weaving -->
    <context:load-time-weaver/>
</beans>
```