title: Spring Data JPA & EntityManager
date: 2016-02-15 17:55:48
tags:
- Spring Data
- JPA
---


## 核心术语

### Persistence Contexts

Persistence Unit 是实体类的一个命名配置。Persistence Context是托管的实体实例的集合。 每个Persistence Context关联一个Persistence Unit，限制了托管的实例的类在Persistence Unit所定义的类的集合范围内。一个实体实例被托管的意思是说，它被包含在一个Persistence Context中并且能够通过Entity Manager来操纵它。 因此，我们说Entity Manager管理一个Persistence Context。

如果一个Persistence Context参与到事务中，那么托管的实体的在内存中的状态将会同步到数据库。

<!--more-->

### Entity Managers

JPA定义了三种类型的Entity Manager。

#### Container-Managed Entity Managers

在Java EE环境下，获取Entity Manager最通用的方式就是通过`@PersistenceContext`注解，这种方式获得的Entity Manager叫做*container-managed*，因为容器管理了Entity Manager的生命周期。 应用不需要创建或者关闭它。

Container-managed Entity Managers有两个变种。Container-managed Entity Manager的风格决定了它如何和Persistence Contexts一起工作。

- 第一种也是最常见的叫做*transaction-scoped*。这意味着Entity Manager的管理的Persistence Contexts的作用域由当前的JTA事务决定。
- 第二种叫做*extended*。Extended Entity Managers使用一个绑定到stateful session beans的生命周期的Persistence Context。它的作用域由stateful session bean的生命周期决定，可能跨越多个事务。

#### Application-Managed Entity Managers

任何由调用`EntityManagerFactory`实例的`createEntityManager`方法创建的Entity Manager叫做*application-managed* Entity Manager。


## 事务管理

事务决定了新建、修改、移除的实体何时同步到数据库中。

### 事务类型

#### 全局事务

全局事务能够让你使用多个事务资源，通常是关系型数据库和消息队列。 应用服务器通过JTA管理全局事务。另外，一个JTA的`UserTransaction`通常需要从JDNI获取，意味着想要使用JTA你也必须使用JDNI。

更理想的使用全局事务的方式是通过EJB CMT (Container Managed Transaction): CMT是一种声明式事务管理（区别于编程式事务管理）。EJB CMT 移除了事务相关的JDNI查找的需要，尽管EJB本身还是需要使用JDNI的。它还移除了书写Java代码来控制事务的需要，但不是全部。CMT重要的缺陷是，它依赖于JTA和应用服务器环境。

#### 本地事务

本地事务是特定于资源的，比如一个关联于JDBC连接的事务。本地事务使用起来很简单，但是重要的缺陷在于不能跨事务资源。

### JTA事务管理

为了讨论JTA事务，我们必须先讨论*transaction synchronization*、*transaction association*以及*transaction propagation*的区别。 

- *Transaction synchronization*是Persistence Context注册一个事务的过程，这样当事务提交时，Persistence Context能够被通知到。JPA提供商使用这个通知来确保Persistence Context被正确的同步到数据库。
- *Transaction association*是绑定Persistence Context到事务的行为。你可以把它当做事务作用范围内的活动的Persistence Context。
- *Transaction propagation*是在一个事务中，多个Container-managed Entity Managers共享一个Persistent Context的过程。

在一个JTA事务中，只有一个Persistent Context能够被关联和传播。一个事务中的所有的Container-managed Entity Managers必须共享同一个被传播的Persistent Context。

#### Transaction-Scoped Persistence Contexts

Transaction-scoped Persistence Context被绑定到事务的生命周期。在事务期间它由容器创建，并且当事务结束时被关闭。当需要时，Transaction-scoped Entity Managers负责自动创建Transaction-scoped Persistence Contexts。 这里说当需要时，是因为Transaction-scoped Persistence Context的创建是惰性的。 只有当Entity Manager的方法被调用并且没有Persistent Context存在时，Entity Manager才会创建Persistence Context。

当Transaction-scoped Entity Manager上的一个方法被调用时，它首先查看是否有一个传播的Persistence Context。如果存在，Entity Manager使用那个Persistent Context进行操作。 如果不存在，Entity Manager请求一个新的Persistent Context，同时标记它为当前事务的传播的Persistence Context。所有后续的Transaction-scoped Entity Manager的操作，都会使用这个新创建的Persistent Context。 这种行为独立工作，不管是使用了容器管理的事务还是Bean管理的事务。


#### Extended Persistence Contexts

Extended Persistence Context的生命周期绑定到它关联的stateful session bean。和Transaction-scoped Entity Manager为每个事务创建新的Persistent Context不同，Extended Entity Manager一直使用同一个Persistent Context。Stateful session bean关联到一个唯一的Extended Persistence Context。这个Extended Persistence Context在bean实例创建的时候被创建，当bean实例删除的时候被关闭。 这对Extended Persistence Context的关联和传播特性都有影响。

Extended Persistence Contexts的Transaction association是立即进行的。 使用容器事管理的情况，只要bean的方法被调用，容器自动地关联Persistent Context到事务。同样的，Bean管理的事务的情况，只要`UserTransaction.begin()`被调用，容器拦截到调用就会进行事务关联。

因为Transaction-scoped Entity Manager在创建新的Persistent Context之前会使用已经存在的和事务关联的Persistent Context，所以把Extended Persistence Context和Transaction-scoped Entity Managers共享是可能的。 只要Extended Persistence Context在Transaction-scoped Entity Managers被访问之前被传播，它就能被共享。

##### Persistence Context Collision

我们之前说过只有一个Persistent Context能够通过JTA事务传播，也说过Extended Persistent Context总会尝试使自己成为活动的Persistent Context。一个使用了Transaction-scoped Entity Manager的stateless session bean创建了一个新的Persistent Context，然后调用了一个使用了Extended Persistence Context的stateful session bean的方法。 在Extended Persistence Context的迫切关联的阶段，容器将会查找是否有一个活动的Persistent Context。如果有，它必须和它即将要关联的Persistent Context是同一个，否则会抛出异常（和只有一个Persistent Context能够通过JTA事务传播冲突）。

#### Application-Managed Persistence Contexts

和Container-managed Persistence Contexts一样，Application-managed Persistence Contexts能够和JTA事务同步。Persistent Context和事务同步意味着如果事务提交，将会发生一次flush，但是Persistent Context不会被任何Container-managed Entity Managers关联。能够和事务同步的Container-managed Entity Managers没有数量限制，但是只有一个Container-managed Persistence Context能够被关联。

Application-managed Entity Manager有两种方式参与到JTA事务中。如果Persistent Context是在事务中创建的，持久化提供商会自动的将Persistent Context和事务同步。如果Persistent Context在事务之前创建，Persistent Context通过手动调用`EntityManager`接口的`joinTransaction`来同步事务。一旦同步后，当事务提交时，Persistent Context会自动flush。

### Unsynchronized Persistence Contexts

在普通环境下，JTA Entity Manager和JTA事务同步并且当事务提交后会保存所有的修改。 但是有些时候，需要像Application-managed Entity Manager的`joinTransaction`一样，手动的实现事务同步。为了获取一个包含Unsynchronized Persistence Context的Container-managed Entity Manager，@PersistenceContext注解的synchronization属性来实现

```
@PersistenceContext(unitName="EmployeeService",synchronization=UNSYNCHRONIZED)
EntityManager em;
```

### 本地事务

本地事务由应用程序显式的控制。应用通过从Entity Manager获取`javax.persistence.EntityTransaction`的实现来和本地事务交互。

```
public interface EntityTransaction {
    public void begin();
    public void commit();
    public void rollback();
    public void setRollbackOnly();
    public boolean getRollbackOnly();
    public boolean isActive();
}
```

```
public class ExpirePasswords {
    public static void main(String[] args) {
        int maxAge = Integer.parseInt(args[0]);
        String defaultPassword = args[1];
        EntityManagerFactory emf =
            Persistence.createEntityManagerFactory("admin");
        try {
            EntityManager em = emf.createEntityManager();
            Calendar cal = Calendar.getInstance();
            cal.add(Calendar.DAY_OF_YEAR, -maxAge);
            em.getTransaction().begin();
            List<User> expired =
                em.createQuery("SELECT u FROM User u WHERE u.lastChange <= ?1", User.class)
                  .setParameter(1, cal, TemporalType.DATE)
                  .getResultList();
            for (User u : expired) {
                System.out.println("Expiring password for " + u.getName());
                u.setPassword(defaultPassword);
            }
            em.getTransaction().commit();
            em.close();
        } finally {
            emf.close();
        } 
    }
}
```

## Spring Data JPA

Spring Data JPA中Entity Manager的实现类似于Transaction-Scoped Persistence Contexts。 下面是`JpaTransactionManager`部分源码

事务开始时，如果没有关连的Entity Manager，则会创建新的Entity Manager

```
@Override
protected void doBegin(Object transaction, TransactionDefinition definition) {

	……

	try {
		if (txObject.getEntityManagerHolder() == null ||
				txObject.getEntityManagerHolder().isSynchronizedWithTransaction()) {
			EntityManager newEm = createEntityManagerForTransaction();
			if (logger.isDebugEnabled()) {
				logger.debug("Opened new EntityManager [" + newEm + "] for JPA transaction");
			}
			txObject.setEntityManagerHolder(new EntityManagerHolder(newEm), true);
		}

		EntityManager em = txObject.getEntityManagerHolder().getEntityManager();

	……
}

事务结束时，关闭Entity Manager

protected void doCleanupAfterCompletion(Object transaction) {
	JpaTransactionObject txObject = (JpaTransactionObject) transaction;

	// Remove the entity manager holder from the thread, if still there.
	// (Could have been removed by EntityManagerFactoryUtils in order
	// to replace it with an unsynchronized EntityManager).
	if (txObject.isNewEntityManagerHolder()) {
		TransactionSynchronizationManager.unbindResourceIfPossible(getEntityManagerFactory());
	}
	txObject.getEntityManagerHolder().clear();

	……

	// Remove the entity manager holder from the thread.
	if (txObject.isNewEntityManagerHolder()) {
		EntityManager em = txObject.getEntityManagerHolder().getEntityManager();
		if (logger.isDebugEnabled()) {
			logger.debug("Closing JPA EntityManager [" + em + "] after transaction");
		}
		EntityManagerFactoryUtils.closeEntityManager(em);
	}
	else {
		logger.debug("Not closing pre-bound JPA EntityManager after transaction");
	}
}
```

另外`OpenEntityManagerInViewInterceptor`实现了"Open EntityManager in View"的模式，解决了实体对象在视图中懒加载的问题。它会在Controller执行之前就创建Entity Manager，最后在视图渲染完成后关闭Entity Manager。

```
public void preHandle(WebRequest request) throws DataAccessException {

    ……

	if (TransactionSynchronizationManager.hasResource(getEntityManagerFactory())) {
		// Do not modify the EntityManager: just mark the request accordingly.
		Integer count = (Integer) request.getAttribute(participateAttributeName, WebRequest.SCOPE_REQUEST);
		int newCount = (count != null ? count + 1 : 1);
		request.setAttribute(getParticipateAttributeName(), newCount, WebRequest.SCOPE_REQUEST);
	}
	else {
		logger.debug("Opening JPA EntityManager in OpenEntityManagerInViewInterceptor");
		try {
			EntityManager em = createEntityManager();
			EntityManagerHolder emHolder = new EntityManagerHolder(em);
			TransactionSynchronizationManager.bindResource(getEntityManagerFactory(), emHolder);

			AsyncRequestInterceptor interceptor = new AsyncRequestInterceptor(getEntityManagerFactory(), emHolder);
			asyncManager.registerCallableInterceptor(participateAttributeName, interceptor);
			asyncManager.registerDeferredResultInterceptor(participateAttributeName, interceptor);
		}
		catch (PersistenceException ex) {
			throw new DataAccessResourceFailureException("Could not create JPA EntityManager", ex);
		}
	}
}


public void afterCompletion(WebRequest request, Exception ex) throws DataAccessException {
	if (!decrementParticipateCount(request)) {
		EntityManagerHolder emHolder = (EntityManagerHolder)
				TransactionSynchronizationManager.unbindResource(getEntityManagerFactory());
		logger.debug("Closing JPA EntityManager in OpenEntityManagerInViewInterceptor");
		EntityManagerFactoryUtils.closeEntityManager(emHolder.getEntityManager());
	}
}

```

