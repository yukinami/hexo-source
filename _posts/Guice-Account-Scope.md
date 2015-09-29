title: Guice Account Scope
tags:
  - Guice
  - Android
  - Spring Social
date: 2015-02-02 22:43:52
---

Google Guice 支持自定义Scope，当然通常情况下我们不需要自己实现Scope。RoboGuice作为Android的IOC容器，实现了基于当前Context的ContextScope。我们可以根据自己需求自定义Scope，实现一些特殊的需求。GitHub的Android客户端定义了一个AccountScope，实现了基于当前账户的上下文对象的注入。
<!--more-->
自定义Scope
===========

Scope接口
--------
Scope接口本身很简单
```
public interface Scope {

  /**
   * Scopes a provider. The returned provider returns objects from this scope.
   * If an object does not exist in this scope, the provider can use the given
   * unscoped provider to retrieve one.
   *
   * <p>Scope implementations are strongly encouraged to override
   * {@link Object#toString} in the returned provider and include the backing
   * provider's {@code toString()} output.
   *
   * @param key binding key
   * @param unscoped locates an instance when one doesn't already exist in this
   *  scope.
   * @return a new provider which only delegates to the given unscoped provider
   *  when an instance of the requested object doesn't already exist in this
   *  scope
   */
  public <T> Provider<T> scope(Key<T> key, Provider<T> unscoped);

  /**
   * A short but useful description of this scope.  For comparison, the standard
   * scopes that ship with guice use the descriptions
   * {@code "Scopes.SINGLETON"}, {@code "ServletScopes.SESSION"} and
   * {@code "ServletScopes.REQUEST"}.
   */
  String toString();
}
```

scope方法需要实现，如何根据指定的Key，返回其对应的Provider。通常直接返回`Provider`匿名实现类实例，在其实现方法`get`方法中，具体实现如何scope对应的对象。

实现接口
-------
接口实现参考官方示例

```
/**
 * Scopes a single execution of a block of code. Apply this scope with a
 * try/finally block: <pre><code>
 *
 *   scope.enter();
 *   try {
 *     // explicitly seed some seed objects...
 *     scope.seed(Key.get(SomeObject.class), someObject);
 *     // create and access scoped objects
 *   } finally {
 *     scope.exit();
 *   }
 * </code></pre>
 *
 * The scope can be initialized with one or more seed values by calling
 * <code>seed(key, value)</code> before the injector will be called upon to
 * provide for this key. A typical use is for a servlet filter to enter/exit the
 * scope, representing a Request Scope, and seed HttpServletRequest and
 * HttpServletResponse.  For each key inserted with seed(), you must include a
 * corresponding binding:
 *  <pre><code>
 *   bind(key)
 *       .toProvider(SimpleScope.&lt;KeyClass&gt;seededKeyProvider())
 *       .in(ScopeAnnotation.class);
 * </code></pre>
 *
 * @author Jesse Wilson
 * @author Fedor Karpelevitch
 */
public class SimpleScope implements Scope {

  private static final Provider<Object> SEEDED_KEY_PROVIDER =
      new Provider<Object>() {
        public Object get() {
          throw new IllegalStateException("If you got here then it means that" +
              " your code asked for scoped object which should have been" +
              " explicitly seeded in this scope by calling" +
              " SimpleScope.seed(), but was not.");
        }
      };
  private final ThreadLocal<Map<Key<?>, Object>> values
      = new ThreadLocal<Map<Key<?>, Object>>();

  public void enter() {
    checkState(values.get() == null, "A scoping block is already in progress");
    values.set(Maps.<Key<?>, Object>newHashMap());
  }

  public void exit() {
    checkState(values.get() != null, "No scoping block in progress");
    values.remove();
  }

  public <T> void seed(Key<T> key, T value) {
    Map<Key<?>, Object> scopedObjects = getScopedObjectMap(key);
    checkState(!scopedObjects.containsKey(key), "A value for the key %s was " +
        "already seeded in this scope. Old value: %s New value: %s", key,
        scopedObjects.get(key), value);
    scopedObjects.put(key, value);
  }

  public <T> void seed(Class<T> clazz, T value) {
    seed(Key.get(clazz), value);
  }

  public <T> Provider<T> scope(final Key<T> key, final Provider<T> unscoped) {
    return new Provider<T>() {
      public T get() {
        Map<Key<?>, Object> scopedObjects = getScopedObjectMap(key);

        @SuppressWarnings("unchecked")
        T current = (T) scopedObjects.get(key);
        if (current == null && !scopedObjects.containsKey(key)) {
          current = unscoped.get();

          // don't remember proxies; these exist only to serve circular dependencies
          if (Scopes.isCircularProxy(current)) {
            return current;
          }

          scopedObjects.put(key, current);
        }
        return current;
      }
    };
  }

  private <T> Map<Key<?>, Object> getScopedObjectMap(Key<T> key) {
    Map<Key<?>, Object> scopedObjects = values.get();
    if (scopedObjects == null) {
      throw new OutOfScopeException("Cannot access " + key
          + " outside of a scoping block");
    }
    return scopedObjects;
  }

  /**
   * Returns a provider that always throws exception complaining that the object
   * in question must be seeded before it can be injected.
   *
   * @return typed provider
   */
  @SuppressWarnings({"unchecked"})
  public static <T> Provider<T> seededKeyProvider() {
    return (Provider<T>) SEEDED_KEY_PROVIDER;
  }
}
```

这个实现类实现了基于ThreadLocal，当前线程上下文的依赖注入。

定义Scope
---------
上面只是实现了Scope接口，对于这个实现，需要进行定义其注解，并且绑定到注解

###定义注解
```
@Target({ TYPE, METHOD }) @Retention(RUNTIME) @ScopeAnnotation
public @interface BatchScoped {}
```

###绑定到注解
```
public class BatchScopeModule {
  public void configure() {
    SimpleScope batchScope = new SimpleScope();

    // tell Guice about the scope
    bindScope(BatchScoped.class, batchScope);

    // make our scope instance injectable
    bind(SimpleScope.class)
        .annotatedWith(Names.named("batchScope"))
        .toInstance(batchScope);
  }
}
```

触发Scope
---------
```
@Inject @Named("batchScope") SimpleScope scope;

  /**
   * Runs {@code runnable} in batch scope.
   */
  public void scopeRunnable(Runnable runnable) {
    scope.enter();
    try {
      // explicitly seed some seed objects...
      scope.seed(Key.get(SomeObject.class), someObject);

      // create and access scoped objects
      runnable.run();

    } finally {
      scope.exit();
    }
  }
```

`SimpleScope`需要先调用`enter`进入Scope，然后即可进行创建，注入Scoped对象。这样的代码比较底层，通常在过滤器或者拦截器中实现Scope的enter和exit。

实现AccountScope
================
和上面实现的基于Thread的Scope类似，首先我们需要一个容器来维护所有Account的对象。这里可以用一个二维Map实现对应的保存，ThreadLocal保存当前线程所在的Account

```
ThreadLocal<Account> currentAccount = new ThreadLocal<Account>();

Map<NoodlesAccount, Map<Key<?>, Object>> scopeMaps = new ConcurrentHashMap<Account, Map<Key<?>, Object>>();
```

然后需要实现基于Account上下文的注入时，将当前ThreadLocal的值设置为对应的Account对象，`scope`方法的实现中即可根据当前线程上下文的Account找到对应的`scopeMaps`的value值。

```
public void enterWith(final NoodlesAccount account) {
    if (currentAccount.get() != null)
        throw new IllegalStateException(
                "A scoping block is already in progress");

    currentAccount.set(account);
}
```
```
protected <T> Map<Key<?>, Object> getScopedObjectMap(final Key<T> key) {
    NoodlesAccount account = currentAccount.get();
    if (account == null)
        throw new IllegalStateException("Cannot access " + key
                + " outside of a scoping block");

    Map<Key<?>, Object> scopeMap = scopeMaps.get(account);
    if (scopeMap == null) {
        scopeMap = new ConcurrentHashMap<Key<?>, Object>();
        scopeMap.put(NOODLES_ACCOUNT_KEY, account);
        scopeMaps.put(account, scopeMap);
    }
    return scopeMap;
}

public <T> void seed(Key<T> key, T value) {
    Map<Key<?>, Object> scopedObjects = getScopedObjectMap(key);
    Preconditions.checkState(!scopedObjects.containsKey(key), "A value for the key %s was " +
                    "already seeded in this scope. Old value: %s New value: %s", key,
            scopedObjects.get(key), value);
    scopedObjects.put(key, value);
}
```

完整的实现可以[参考][2]。

使用AccountScope
================
Android平台下，需要使用到AccountScope的地方，比如说Account的账户数据同步；对API接口的访问。

Scope API接口的访问
------------------
对API接口的访问通常在AsyncLoader或者AsyncTask中进行，因此只对这两个类进行一定抽象可以实现Scope API接口。

在`loadInBackground`中实现如下
```
accountScope.enterWith(account);
try {
    contextScope.enter(getContext());
    try {
        return load(account);
    } finally {
        contextScope.exit(getContext());
    }
} finally {
    accountScope.exit();
}
```
然后即可根据当前Scope的Account对象，获取其Access Token（Android Account Framework），从而实现Scope对API接口的访问。

***TIPS***，这里也同时需要进入Context Scope，注入Context上下文环境中的对象。`ContextScope`的实现使用了ThreadLocal，并且
Async是在其他Thread中执行的，所有必须重新enter，否则无法注入Context上下文中的对象。


集成Spring Social
=================
Spring Social是以API Binding的方式[实现][3]API的访问。结合Roboguice的，理想的实现便是进行一次Binding，并将Binding对象注入guice容器中，然后在需要是要使用直接注入Binding对象即可。但是Spring Social绑定的过程中即需要access token，而上面对AccountScope的使用，只有调用AsyncLoader或者AsyncTask中才会进入AccountScope，获取其access token，这样只能每次调用的时候都进行一次API Binding。 所以可以对Spring Social进行扩展，使其Binding的时候不再依赖access token，而是依赖于Account的Provider对象。

Provider
--------
Provider除了可以用于封装对象相对复杂的初始化，更重要的时可以实现Lazy loading。 即在初始化不直接注入对象本身，而是注入对应的Provider，然后在使用的再调用`get`方法获取实例。

因为注入的Provider是个代理对象，最终会代理给直接Provider的对象返回结果，所以可以通过先bind一个空的Provider实现，然后进入对应的Scope后，注入实际的Provider。

```
private static final Provider<Object> SEEDED_KEY_PROVIDER = new Provider<Object>() {
    public Object get() {
        throw new IllegalStateException("Object not seeded in this scope");
    }
};

public static <T> Provider<T> seededKeyProvider() {
    return (Provider<T>) SEEDED_KEY_PROVIDER;
}

bind(NOODLES_ACCOUNT_KEY).toProvider(
    AccountScope.<NoodlesAccount>seededKeyProvider()).in(
    scope);
```

扩展API Binding
----------------
利用Provider的机制，扩展API Binding，不再依赖access token，而是依赖Account的Provider，即可较为完美解决之前的问题。

RoboNoodlesServiceProvider覆盖getApi方法，返回依赖于Account的template
```
@Override
public Noodles getApi(String accessToken) {
    if (AUTH_TOKEN.equals(accessToken)) {
        return getApi(account);
    }
    return new NoodlesTemplate(accessToken);
}

public Noodles getApi(Provider<NoodlesAccount> account) {
    return new RoboNoodlesTemplate(account);
}
```

Template类重新配置Interceptor
```
public class RoboNoodlesTemplate extends NoodlesTemplate {

    private Provider<NoodlesAccount> account;

    public RoboNoodlesTemplate(Provider<NoodlesAccount> account) {
        this.account = account;
        configureRestTemplate();
    }

    protected void configureRestTemplate() {
        RestTemplate restTemplate = getRestTemplate();
        restTemplate.getInterceptors().add(new RoboOAuth2RequestInterceptor(account, OAuth2Version.BEARER));
    }
}

public class RoboOAuth2RequestInterceptor implements ClientHttpRequestInterceptor {

    private Provider<NoodlesAccount> account;

    private final OAuth2Version oauth2Version;

    public RoboOAuth2RequestInterceptor(Provider<NoodlesAccount> account, OAuth2Version oauth2Version) {
        this.account = account;
        this.oauth2Version = oauth2Version;
    }

    public ClientHttpResponse intercept(final HttpRequest request, final byte[] body, ClientHttpRequestExecution execution) throws IOException {
        HttpRequest protectedResourceRequest = new HttpRequestDecorator(request);
        protectedResourceRequest.getHeaders().set("Authorization", oauth2Version.getAuthorizationHeaderValue(account.get().getAuthToken()));
        return execution.execute(protectedResourceRequest, body);
    }
}
```



[1]: https://github.com/google/guice/wiki/CustomScopes
[2]: https://gist.github.com/yukinami/230dceea400d6548e7b3
[3]: http://yukinami.github.io/2015/01/06/%E7%94%A8spring-social%E6%9E%84%E5%BB%BAAPI%E7%9A%84client%E7%AB%AF/
