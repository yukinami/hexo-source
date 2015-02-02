title: 基于Spring的多租户系统的简单实现
tags:
  - Spring Framework
date: 2014-11-11 17:44:39
---

### 前言

对于一个完整的将web 应用程序转换为 SaaS 应用程序的过程而言，需要满足以下7个条件：

1.  应用程序必须支持多租户
2.  应用程序必须具备某种程度的自助注册功能。
3.  必须具备订阅/记账机制。
4.  应用程序必须能够有效地扩展。
5.  必须能够监视、配置和管理应用程序和租户。
6.  必须有一种机制能够支持惟一的用户标识和身份验证。
7.  必须有一种机制能够支持对每个租户进行某种程度的自定义。

本文主要讨论的是实现SaaS的核心，支持多租户。

<!--more-->

### 理论基础

实现多租户的方式，大致分为三种

*   单独的数据库

[![](http://yukinamia-wordpress.stor.sinaapp.com/uploads/2014/11/IC6354-300x107.gif "IC6354")](http://yukinamia-wordpress.stor.sinaapp.com/uploads/2014/11/IC6354.gif)

将租户的数据分离到单独的数据库需要较高的硬件成本和维护成本，但是数据的隔离性更好，安全性更高。

*   共享的数据库，单独的Schema

[![](http://yukinamia-wordpress.stor.sinaapp.com/uploads/2014/11/IC106518-293x300.gif "IC106518")](http://yukinamia-wordpress.stor.sinaapp.com/uploads/2014/11/IC106518.gif)

这种方式减少了一定的成本，并且也拥有较好的逻辑数据隔离性。但是当数据库崩溃的时候较难恢复，恢复一个租户的数据需要恢复整个数据库，意味着不管其他的租户数据有没有失败，所有的数据都会被覆盖。

*   共享的数据库，共享的Schema

[![](http://yukinamia-wordpress.stor.sinaapp.com/uploads/2014/11/IC216.gif "IC216")](http://yukinamia-wordpress.stor.sinaapp.com/uploads/2014/11/IC216.gif)

这种方式的硬件成本和维护成本是最小的。但是所有的压力都聚集到了应用这端，数据的隔离，安全性等问题都需要应用端来处理。

### 实现思路

本文主要针对第三种实现方式，在应用层实现多租户的功能，具体的实现是基于spring framework的。

应用层实现多租户的重点，需要解决以下两个问题：

1.  用户所属租户的验证。用户所属的租户决定了用户所访问的数据。具体的实现可以是通过用户登录验证机制，查找用户所属的租户；或者是通过URL的子域，path，参数等等绑定到租户，通过过滤器设置用户所属的租户。
2.  租户数据的隔离和访问。租户对数据库的访问方式决定了数据表的隔离方式。比如说，我们可以让不同的租户使用不同的数据库用户来访问数据库，在数据库层对需要租户隔离的表创建动态视图，动态视图的条件就是TANANTID等于当前的数据库用户的租户ID。如果不同用户的数据库连接是隔离的，那么都不用不同的数据库，直接在设置connection的session用户变量为租户名或者租户ID，再进行动态视图的隔离。

### 具体实现

#### 用户所属租户的验证

本文使用Servletpath绑定到租户的方式来实现租户的验证。

容器启动的时候，注册所有的租户的business name(租户的一个标示)绑定到servlet mapping中：
<pre class="crayon-plain-tag">@Bean(name = DispatcherServletAutoConfiguration.DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
    public ServletRegistrationBean dispatcherServletRegistration() {
        Iterable&lt;Tenant&gt; tenants = this.tenantRepository.findAll();

        ServletRegistrationBean registration = new ServletRegistrationBean(
                dispatcherServlet(), getServletMappings(tenants));
        registration.setName(DispatcherServletAutoConfiguration.DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
        if (this.multipartConfig != null) {
            registration.setMultipartConfig(this.multipartConfig);
        }

        return registration;
    }</pre>

可以对映射的mapping进行一定的混淆工作，避免租户间恶意的访问。然后再实现一个拦截器将访问请求的servlet path解析为business name：
<pre class="crayon-plain-tag">@Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String businessName = resolve(urlPathHelper.getServletPath(request));
        request.setAttribute(TENANT_BUSINESS_NAME_KEY, businessName);

        //TODO other process
}</pre>

#### 数据库的数据隔离

对所有的需要租户隔离的业务表添加tenant_dbu(租户的数据库用户)，tenant_id(租户的ID)。然后创建动态视图，获取当前的数据库用户：
<pre class="crayon-plain-tag">CREATE TABLE inventory (
  id BIGINT IDENTITY PRIMARY KEY,
  name VARCHAR(55),
  tenant_dbu VARCHAR(16) NOT NULL ,
  tenant_id BIGINT NOT NULL
);</pre><pre class="crayon-plain-tag">CREATE VIEW inventory_vw AS
SELECT id, name
FROM inventory
WHERE tenant_dbu = CURRENT_USER;</pre>

同时再实现一个触发器，再往租户业务表中插入的时候，插入正确的当前的数据库用户。
<pre class="crayon-plain-tag">CREATE TRIGGER tr_inventory_before_insert
BEFORE INSERT ON inventory
REFERENCING NEW AS newrow FOR EACH ROW
BEGIN ATOMIC
IF (CURRENT_USER = 'root') THEN
  SET newrow.tenant_dbu = CURRENT_USER;
END IF;
END</pre>

#### 应用对数据隔离的实现

上述在数据库层简单地实现了对数据的隔离，最终还需要进行正确的操作，才能保证访问到正确的数据。

这里需要实现一个能够进行动态路由的数据源，不同的租户使用不同的数据库用户的链接。

这里使用Spring提供的AbstractRoutingDataSource。首先需要正确的初始化所有的数据源：
<pre class="crayon-plain-tag">protected void registerDataSource(Iterable&lt;Tenant&gt; tenants) {
        Map&lt;Object, Object&gt; targetDataSources = new HashMap&lt;&gt;();
        for (Tenant tenant : tenants) {

            DataSourceBuilder factory = DataSourceBuilder
                    .create(this.properties.getClassLoader())
                    .url(this.properties.getUrl())
                    .username(tenant.getDbu())
                    .password(tenant.getEdbpwd());

            targetDataSources.put(tenant.getBusinessName(), factory.build());
        }

        ((AbstractRoutingDataSource) this.dataSource).setTargetDataSources(targetDataSources);
        ((AbstractRoutingDataSource) this.dataSource).afterPropertiesSet();
    }</pre>

在用户进行访问的时候，设置当前租户的business name，以进行对数据源的路由：
<pre class="crayon-plain-tag">@Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String businessName = (String)request.getAttribute(TenantResolveInterceptor.TENANT_BUSINESS_NAME_KEY);
        TenantContextHolder.setBusinessName(businessName);
        return true;
    }</pre><pre class="crayon-plain-tag">public class TenantContextHolder {

    private static final ThreadLocal&lt;String&gt; contextHolder =
            new ThreadLocal&lt;String&gt;();

    public static void setBusinessName(String businessName) {
        Assert.notNull(businessName, "businessName cannot be null");
        contextHolder.set(businessName);
    }

    public static String getBusinessName() {
        return contextHolder.get();
    }

    public static void clearBusinessName() {
        contextHolder.remove();
    }
}</pre>

#### 其他的一些优化

系统中有一些资源是属于各个用户的，有一些资源的系统级别的。我们不希望租户访问到一些系统页面，也不希望系统访问一些租户数据。可以通过拦截器进行一定的拦截：
<pre class="crayon-plain-tag">// restrict the access
        HandlerMethod method = (HandlerMethod) handler;
        TenantResource tenantResource = method.getMethodAnnotation(TenantResource.class);
        RootResource rootResource = method.getMethodAnnotation(RootResource.class);

        boolean isRootResource = false;

        // get annotation from class when no annotation is specified
        if (tenantResource == null &amp;&amp; rootResource == null) {
            tenantResource = AnnotationUtils.findAnnotation(method.getBeanType(), TenantResource.class);
            rootResource = AnnotationUtils.findAnnotation(method.getBeanType(), RootResource.class);
        }

        // still with no annotation, set default
        if (tenantResource == null &amp;&amp; rootResource == null) {
            isRootResource = true;
        }

        // tenant resource
        if (tenantResource != null &amp;&amp; StringUtils.isEmpty(businessName)) {
            throw new NoHandlerFoundException(request.getMethod(), request.getRequestURI(), null);
        }

        // root resource
        if ((rootResource != null || isRootResource) &amp;&amp; !StringUtils.isEmpty(businessName)) {
            throw new NoHandlerFoundException(request.getMethod(), request.getRequestURI(), null);
        }</pre>

我们自己是实现了两个注解@RootResource，@TenantResource，用来标记资源是租户级别的还是系统级别的。

完整的示例请[参考](https://github.com/yukinami/spring-boot-saas)。

### 写在最后

本文只是抛砖引玉，简单地实现了基于SaaS的系统的部分功能，期待您的反馈。