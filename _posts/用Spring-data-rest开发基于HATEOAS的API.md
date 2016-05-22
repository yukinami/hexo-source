title: 用Spring data rest开发基于HATEOAS的API
date: 2016-05-22 10:17:03
tags:
- API
- Sring Data Rest
---

## 概念

首先，HATEOAS (Hypermedia as the Engine of Application State) 是REST应用架构的一个约束。一个*hypermedia-driven*的站点通过响应中的超媒体链接动态地提供了导航到站点的REST接口的信息。

下面是一个基于HATEOAS的响应

```
{
    "name": "Alice",
    "links": [ {
        "rel": "self",
        "href": "http://localhost:8080/customer/1"
    } ]
}
```
<!-- more -->

Spring使用HAL作为HATEOAS的实现。

Spring Data Rest是Spring Data的子项目，那么该项目是为了实现类似Spring Data JPA的统一数据访问层接口的目的。 实际情况，只要定义了Spring Data的标准Repository接口，那么Spring Data Rest便会为你提供一套标准的基于HTEOAS的REST接口。 如果你的应用架构是基于DDD的，那么对Spring Data Rest接口接入会显得非常友好。

## Spring对于HAL的实现

使用Spring Data Rest时，`RepositoryRestMvcConfiguration`中注册的jacksonHttpMessageConverter是负责把ResourceSupport子类对象渲染成HAL格式的JSON字符串的。通常情况下对于请求的Media Type是`application/hal+json`时，才会用这个converter进行转化。`useHalAsDefaultJsonMediaType`可以控制，当请求JSON media type时，是否默认使用HAL。这个参数的默认值是true，也就是说如果客户端请求的是普通的application/json，对于ResourceSupport子类对象依然可以返回HAL格式的JSON。 另外如果请求时不指定Media Type的，那么Spring Data Rest的defaultMediaType配置将会生效，默认值为`application/hal+json`。

下面是注册jacksonHttpMessageConverter的相关代码

```
@Bean
public TypeConstrainedMappingJackson2HttpMessageConverter halJacksonHttpMessageConverter() {

	ArrayList<MediaType> mediaTypes = new ArrayList<MediaType>();
	mediaTypes.add(MediaTypes.HAL_JSON);

	// Enable returning HAL if application/json is asked if it's configured to be the default type
	if (config().useHalAsDefaultJsonMediaType()) {
		mediaTypes.add(MediaType.APPLICATION_JSON);
	}

	int order = config().useHalAsDefaultJsonMediaType() ? Ordered.LOWEST_PRECEDENCE - 10
			: Ordered.LOWEST_PRECEDENCE - 1;

	TypeConstrainedMappingJackson2HttpMessageConverter converter = new ResourceSupportHttpMessageConverter(order);
	converter.setObjectMapper(halObjectMapper());
	converter.setSupportedMediaTypes(mediaTypes);

	return converter;
}
```

如果没有使用Spring Data Rest而是单独使用Spring HATOAS的话，这个jacksonHttpMessageConverter将由`HypermediaSupportBeanDefinitionRegistrar`来注册。 Spring Boot的情况，`HypermediaAutoConfiguration`会导入`HypermediaHttpMessageConverterConfiguration`来针对`spring.hateoas.use-hal-as-default-json-media-type`配置来支持`application/json`。

[这里][1]是关于Spring HATEOAS和Spring Data Rest的自动配置的一些说明。在这次修复之前，由于`RepositoryRestMvcAutoConfiguration`会早于`JacksonAutoConfiguration`运行，导致`JacksonAutoConfiguration`被间接的关闭，没有注册@Primary的ObjectMapper，从而导致注入到`JacksonHttpMessageConvertersConfiguration`的ObjectMapper是一个被HAL全局污染的ObjectMapper。

## 使用中可能会遇到的一些问题

下面的问题都是针对Spring Data Rest配合Spring Data JPA一起使用，并使用Hibernate作为JPA的Vendor。

### 自定义方法实现

有些时候，我们希望覆盖Spring Data Rest的标准实现，或者实现一些额外的接口，就需要自定义处理方法。

- 自定义的Controller需要使用@RepositoryRestController注解，这样才能让Spring Data Rest处理。
- URL的路径必须属于某个Repository的资源路径，下面是`RepositoryRestHandlerMapping`的一段处理逻辑
  ```
  @Override
	protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {

		HandlerMethod handlerMethod = super.lookupHandlerMethod(lookupPath, request);

		if (handlerMethod == null) {
			return null;
		}

		String repositoryLookupPath = new BaseUri(configuration.getBaseUri()).getRepositoryLookupPath(lookupPath);

		// Repository root resource
		if (!StringUtils.hasText(repositoryLookupPath)) {
			return handlerMethod;
		}

		return mappings.exportsTopLevelResourceFor(getRepositoryBasePath(repositoryLookupPath)) ? handlerMethod : null;
	}
  ```
- 注入PersistentEntityResourceAssembler来装配符合HATEOAS的响应JSON对象


### API version的实现

通过上面Spring对HAL的实现中提到的，如果我们想通过请求`application/vnd.xxx.vxx+hal+json`来实现版本的话，jacksonHttpMessageConverter是不会起作用的，因为它只会严格匹配`application/json`和`application/hal+json`。因此那么我们需要扩展匹配的Media Type来实现。

```
@Bean
@DependsOn("halMessageConverterSupportedMediaTypeCustomizer")
public HalVersionMessageConverterSupportedMediaTypesCustomizer registerCustomMediaType(@Qualifier("halJacksonHttpMessageConverter") TypeConstrainedMappingJackson2HttpMessageConverter halJacksonHttpMessageConverter) {
    return new HalVersionMessageConverterSupportedMediaTypesCustomizer();
}

private class HalVersionMessageConverterSupportedMediaTypesCustomizer implements BeanFactoryAware {

    private static final String HAL_JACKSON_HTTP_MESSAGE_CONVERTER_BEAN_NAME = "halJacksonHttpMessageConverter";
    private volatile BeanFactory beanFactory;

    @PostConstruct
    public void customizedSupportedMediaTypes() {
        TypeConstrainedMappingJackson2HttpMessageConverter halJacksonHttpMessageConverter = beanFactory.getBean(HAL_JACKSON_HTTP_MESSAGE_CONVERTER_BEAN_NAME, TypeConstrainedMappingJackson2HttpMessageConverter.class);
        List<MediaType> supportedMediaTypes = new ArrayList<>(halJacksonHttpMessageConverter.getSupportedMediaTypes());
        supportedMediaTypes.add(new MediaType("application", "*+hal+json"));
        halJacksonHttpMessageConverter.setSupportedMediaTypes(supportedMediaTypes);
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
}

```
必须在`HalMessageConverterSupportedMediaTypesCustomizer`之后执行，否则会被其覆盖supportedMediaTypes


### 没有定义Repository的实体的关联的问题

如果实体A包含一个到实体B的关联，查询实体A时，返回的结果会根据实体B的Repository是否存在会有不同。

- 存在B的Repository  那么A到B关联会被处理成link，而返回的A的对象中并不会直接包含B的对象
- 不存在B的Repository 那么由于不存在B的资源的链接，自然不会生成link，并且A对象中会直接包含B对象。 如果在查询A对象时B关联是lazy的，那么这里就会产生额外的查询。所以需要注意，如果包含的关联中是实体不存在Repository时，查询时最好就把这些关联对象fetch出来，否则会对性能产生一定的影响。
- 另外不能定义class级别的@RequestMapping，否则路径会被注册两遍。因为标准的`RequestMappingHandlerMapping`是这样判断是否要注册映射的
  ```
  @Override
	protected boolean isHandler(Class<?> beanType) {
		return ((AnnotationUtils.findAnnotation(beanType, Controller.class) != null) ||
				(AnnotationUtils.findAnnotation(beanType, RequestMapping.class) != null));
	}
  ```

### Excerpt不起作用

官方文档中提到，Excerpt只针对单个资源的请求有效，如果资源是集合，那么Excerpt是不会生效的

> Excerpt projections are NOT applied to single resources automatically. They have to be applied deliberately. Excerpt projections are meant to provide a default preview of collection data, but not when fetching individual resources. 

这是框架中的`RepositoryEntityController`处理的逻辑。`PersistentEntityResourceAssembler`有`toFullResource`和`toResource`方法，前者会忽略Excerpt。`RepositoryEntityController`针对集合的情况会调用`toFullResource`，因而Excerpt是不起作用的。

### 定义了Excerpt的实体的关联对象会产生额外的查询的问题

如果实体A包含一个到实体B的关联，实体B的Repository定义了Excerpt的话，那么虽然最终的返回的对象A中并不包含对象B，只是包含了一个link。但是对象的B的值仍然会被获取。在JPA中懒加载的情况下，这个应该不被加载的关联，就会被触发fetch，而产生额外的查询，对性能产生影响。

下面是`PersistentEntityResourceAssember#doWithAssociation`中相关的代码

```
@Override
public void doWithAssociation(Association<? extends PersistentProperty<?>> association) {

	PersistentProperty<?> property = association.getInverse();

	if (!associationLinks.isLinkableAssociation(property)) {
		return;
	}

	if (!projector.hasExcerptProjection(property.getActualType())) {
		return;
	}

	Object value = accessor.getProperty(association.getInverse());

	if (value == null) {
		return;
	}

	……
}
```

如果实体会被关联的话，所以不要轻易定义Excerpt。

### 如何指定多个Projection

如果返回的一个列中包含A和B两个实体，针对这两个实体都想指定Projection怎么办。虽然URL中只能指定一个projection参数，但是projection是可以重名的，只要他们对应的types即projection针对的实体不一样即可。


## @BasePathAwareController注解的Controller中懒加载出错

`RepositoryRestHandlerMapping`中通过`JpaHelper`在interceptor中添加了`OpenEntityManagerInViewInterceptor`，但是`BasePathAwareHandlerMapping`并没有这个拦截器。

通过

```
@Bean
public MappedInterceptor basePathAwareOpenEntityManagerInViewInterceptor(EntityManagerFactory factory) {
    OpenEntityManagerInViewInterceptor omivi = new OpenEntityManagerInViewInterceptor();
    omivi.setEntityManagerFactory(factory);
    return new MappedInterceptor(new String[]{"/api/**"}, omivi);
}
```
即可在所有的HandlerMapping中添加拦截器，可能和`RepositoryRestHandlerMapping`已经存在的会有点重复，但是并没有什么副作用。

[1]: https://github.com/spring-projects/spring-boot/commit/c55900b43398764d924a485b1244bfba8444eab9
