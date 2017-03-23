title: Spring Security的WebSecurityConfiguration配置及初始化流程
date: 2016-02-14 11:26:31
tags:
- Spring Security
---

## 基于源码的流程分析

下面的源码基于`Spring Security 4.0.3`版本。

### 入口 WebSecurityConfiguration

`WebSecurityConfiguration`的目的是配置`WebSecurity`来创建[FilterChainProxy][FilterChainProxy]

```
@Autowired(required = false)
public void setFilterChainProxySecurityConfigurer(
		ObjectPostProcessor<Object> objectPostProcessor,
		@Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
		throws Exception {
	webSecurity = objectPostProcessor
			.postProcess(new WebSecurity(objectPostProcessor));
	if (debugEnabled != null) {
		webSecurity.debug(debugEnabled);
	}

	Collections.sort(webSecurityConfigurers, AnnotationAwareOrderComparator.INSTANCE);

	Integer previousOrder = null;
	for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
		Integer order = AnnotationAwareOrderComparator.lookupOrder(config);
		if (previousOrder != null && previousOrder.equals(order)) {
			throw new IllegalStateException(
					"@Order on WebSecurityConfigurers must be unique. Order of "
							+ order + " was already used, so it cannot be used on "
							+ config + " too.");
		}
		previousOrder = order;
	}
	for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
		webSecurity.apply(webSecurityConfigurer);
	}
	this.webSecurityConfigurers = webSecurityConfigurers;
}
```

<!--more-->

这里会对WebSecurity套用所有的`SecurityConfigurer`的实例，包括自定义的继承了`WebSecurityConfigurerAdapter`的自定义配置。

这里的套用的过程只是把实例添加到`configurers`属性中

```
public <C extends SecurityConfigurer<O, B>> C apply(C configurer) throws Exception {
	add(configurer);
	return configurer;
}
```

最终在build的过程中会初始化这些配置并套用。

### 配置HttpSecurity

WebSecurityConfigurerAdapter会初始化一个默认的HttpSecurity，同时会调用`protected void configure(HttpSecurity http)`来进行自定义的设置。最终HttpSecurity的实例会作为SecurityFilterChainBuilder传入WebSecurity，以及调用`postBuildAction`来设置FilterSecurityInterceptor。

```
public void init(final WebSecurity web) throws Exception {
	final HttpSecurity http = getHttp();
	web.addSecurityFilterChainBuilder(http).postBuildAction(new Runnable() {
		public void run() {
			FilterSecurityInterceptor securityInterceptor = http
					.getSharedObject(FilterSecurityInterceptor.class);
			web.securityInterceptor(securityInterceptor);
		}
	});
}
```
`WebSecurity`中的`FilterSecurityInterceptor`最终还会用来构建`WebInvocationPrivilegeEvaluator`用于页面标签的权限控制。从目前源码上来看，页面标签的`WebInvocationPrivilegeEvaluator`只会使用最后设置到`WebSecurity`中的`FilterSecurityInterceptor`。如果有多个WebSecurityConfigurerAdapter的子类实例，那么只有最后一个HttpSecurity中的FilterSecurityInterceptor才会生效，其余的会被覆盖。***所以，页面标签的权限控制要注意`HttpSecurity`配置的顺序，`WebSecurityConfigurerAdapter`默认优先度是100***

```
/**
 * Creates the {@link HttpSecurity} or returns the current instance
 *
 * ] * @return the {@link HttpSecurity}
 * @throws Exception
 */
protected final HttpSecurity getHttp() throws Exception {
	if (http != null) {
		return http;
	}

	DefaultAuthenticationEventPublisher eventPublisher = objectPostProcessor
			.postProcess(new DefaultAuthenticationEventPublisher());
	localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);

	AuthenticationManager authenticationManager = authenticationManager();
	authenticationBuilder.parentAuthenticationManager(authenticationManager);
	http = new HttpSecurity(objectPostProcessor, authenticationBuilder,
			localConfigureAuthenticationBldr.getSharedObjects());
	http.setSharedObject(UserDetailsService.class, userDetailsService());
	http.setSharedObject(ApplicationContext.class, context);
	http.setSharedObject(ContentNegotiationStrategy.class, contentNegotiationStrategy);
	http.setSharedObject(AuthenticationTrustResolver.class, trustResolver);
	if (!disableDefaults) {
		// @formatter:off
		http
			.csrf().and()
			.addFilter(new WebAsyncManagerIntegrationFilter())
			.exceptionHandling().and()
			.headers().and()
			.sessionManagement().and()
			.securityContext().and()
			.requestCache().and()
			.anonymous().and()
			.servletApi().and()
			.apply(new DefaultLoginPageConfigurer<HttpSecurity>()).and()
			.logout();
		// @formatter:on
	}
	configure(http);
	return http;
}
```

### 构建Filter

WebSecurity最终会构建Servlet Filter。HttpSecurity的performBuild则最终会构建DefaultSecurityFilterChain对象，然后添加到Filter的securityFilterChains中

```
protected final O doBuild() throws Exception {
	synchronized (configurers) {
		buildState = BuildState.INITIALIZING;

		beforeInit();
		init();

		buildState = BuildState.CONFIGURING;

		beforeConfigure();
		configure();

		buildState = BuildState.BUILDING;

		O result = performBuild();

		buildState = BuildState.BUILT;

		return result;
	}
}
```

```
@Override
protected Filter performBuild() throws Exception {
	Assert.state(
			!securityFilterChainBuilders.isEmpty(),
			"At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. Typically this done by adding a @Configuration that extends WebSecurityConfigurerAdapter. More advanced users can invoke "
					+ WebSecurity.class.getSimpleName()
					+ ".addSecurityFilterChainBuilder directly");
	int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
	List<SecurityFilterChain> securityFilterChains = new ArrayList<SecurityFilterChain>(
			chainSize);
	for (RequestMatcher ignoredRequest : ignoredRequests) {
		securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
	}
	for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
		securityFilterChains.add(securityFilterChainBuilder.build());
	}
	FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
	if (httpFirewall != null) {
		filterChainProxy.setFirewall(httpFirewall);
	}
	filterChainProxy.afterPropertiesSet();

	Filter result = filterChainProxy;
	if (debugEnabled) {
		logger.warn("\n\n"
				+ "********************************************************************\n"
				+ "**********        Security debugging is enabled.       *************\n"
				+ "**********    This may include sensitive information.  *************\n"
				+ "**********      Do not use in a production system!     *************\n"
				+ "********************************************************************\n\n");
		result = new DebugFilter(filterChainProxy);
	}
	postBuildAction.run();
	return result;
}
```


## 相关的一些问题

### Spring Boot的H2ConsoleAutoConfiguration导致页面标签的权限控制不正常

如果Spring Boot启用了H2 Console的话，由于H2ConsoleAutoConfiguration并没有注解`@ConditionalOnMissingBean(WebSecurityConfiguration.class)`，所以即便应用配置了WebSecurityConfiguration的子类，如果没有显示地把`security.basic.enabled`设置成false的话，最终还是会导致`H2ConsoleSecurityConfiguration`配置的套用，`H2ConsoleSecurityConfiguration`的优先度是`SecurityProperties.BASIC_AUTH_ORDER - 10`，比较低，反而容易导致`WebSecurity`的`FilterSecurityInterceptor`被覆盖。

[livereload-extensions]: https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#filter-chain-proxy

