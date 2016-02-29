title: Spring Boot Auto Configuration和HATEOAS、Data REST
date: 2016-02-26 20:34:05
tags:
- Spring Boot
- REST
---

## Spring Boot集成HATEOAS

`JacksonAutoConfiguration`会注册Primary ObjectMapper，这个ObjectMapper通过`spring.jackson.*`来配置。
`JacksonAutoConfiguration`配置完后会注册`HypermediaAutoConfiguration`，`HypermediaAutoConfiguration`中会对Primary ObjectMapper注册`Jackson2HalModule`（由apply-to-primary-object-mapper决定）。

<!--more-->

HypermediaConfiguration

```
private void registerHalModule(ObjectMapper objectMapper) {
	objectMapper.registerModule(new Jackson2HalModule());
	Jackson2HalModule.HalHandlerInstantiator instantiator = new Jackson2HalModule.HalHandlerInstantiator(
			HalObjectMapperConfiguration.this.relProvider,
			HalObjectMapperConfiguration.this.curieProvider);
	objectMapper.setHandlerInstantiator(instantiator);
}
```

Spring Boot使用`Jackson2HalModule`来把HAL相关的ObjectMapper配置适用到上下文环境中的Primary ObjectMapper中。 这允许HAL格式的响应能够发送给*application/json*的请求。这就导致了一个副作用，HAL特定的功能污染了Primary ObjectMapper。

Spring Boot 1.3.0中作了一系列修改来简化ObjectMapper的配置。

需改中提到

> Previously, Boot used Jackson2HalModule to apply the HAL-related
ObjectMapper configuration to the context’s primary ObjectMapper. This
was to allow HAL-formatted responses to be sent for requests accepted
application/json (see gh-2147). This had the unwanted side-effect of
polluting the primary ObjectMapper with HAL-specific functionality.
Furthermore, Jackson2HalModule is an internal of Spring HATEOAS that
@olivergierke has asked us to avoid using.

> This commit replaces the use of Jackson2HalModule with a new approach.
Now, the message converters of any RequestMappingHandlerAdapter beans
are examined and any TypeConstrainedMappingJackson2HttpMessageConverter
instances are modified to support application/json in addition to their
default support for application/hal+json. This behaviour can be disabled
by setting spring.hateoas.use-hal-as-default-json-media-type to false.
This property is named after Spring Data REST’s configuration option
which has the same effect when using Spring Data REST. The new property
replaces the old spring.hateoas.apply-to-primary-object-mapper property.


这次提交使用了一个新方法来替换`Jackson2HalModule`的使用。现在所有的RequestMappingHandlerAdapter的message converters会被检查，并且任何`TypeConstrainedMappingJackson2HttpMessageConverter`实例会被修改来支持*application/json*以及它们默认支持的*application/hal+json*。 这个行为可以通过设置`spring.hateoas.use-hal-as-default-json-media-type`为`false`来关闭。这个属性根据Spring Data REST的配置选项来命名，并且和使用Spring Data REST时有相同的作用。

Spring Data Rest中注册的`TypeConstrainedMappingJackson2HttpMessageConverter`也是这样的逻辑

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

这样的话默认的ObjectMapper就不会被HAL特性污染。`TypeConstrainedMappingJackson2HttpMessageConverter`支持转换的HAL对象（Resource）不管请求的是*application/json*还是*application/hal+json*都能够被正确的转换。

## Spring Boot集成Sring Data Rest

另外，此次修改中还提到

> Previously, when Spring Data REST was on the classpath,
JacksonAutoConfiguration would be switched off resulting in the context
containing multiple ObjectMappers, none of which was primary.

> This commit configures RepositoryRestMvcAutoConfiguration to run after
JacksonAutoConfiguration. This gives the latter a chance to create its
primary ObjectMapper before the former adds its ObjectMapper beans to
the context.

之前当Spring Data REST在classpath中时，JacksonAutoConfiguration会被自动关闭，导致上下文环境包含多个ObjectMapper，并且没有一个是primary的。也就是说`RepositoryRestMvcConfiguration`中注册的ObjectMapper同样也会被`JacksonHttpMessageConvertersConfiguration`中配置的MessageConverter使用，同样可能会导致HAL污染的问题。

这次提交修改成配置`RepositoryRestMvcAutoConfiguration`在`JacksonAutoConfiguration`之后执行。这给了后者在前者添加它的ObjectMapper之前，创建primary ObjectMapper的机会。最终达到了隔离ObjectMapper的目录。

另外Spring Boot中对于ObjectMapper配置的属性也会套用到`RepositoryRestMvcAutoConfiguration`中定义的ObjectMapper

```
class SpringBootRepositoryRestConfigurer extends RepositoryRestConfigurerAdapter {

	@Autowired(required = false)
	private Jackson2ObjectMapperBuilder objectMapperBuilder;

	@Override
	public void configureJacksonObjectMapper(ObjectMapper objectMapper) {
		if (this.objectMapperBuilder != null) {
			this.objectMapperBuilder.configure(objectMapper);
		}
	}
}
```

## 自定义MediaType的问题

在1.3.0之前，请求*application/vnd.xxx.v2.5.hal+json*这样的自定义头，由于HAL污染的原因，并且`MappingJackson2HttpMessageConverter`默认支持*application/json*以及*application/\*+json*是能够正确返回HAL格式的。 现在虽然`TypeConstrainedMappingJackson2HttpMessageConverter`也支持*application/json*，但是由于自定义的头并不符合，所以并不能正确返回HAL格式的响应。


### 解决方案

注册一个带有通配符的MediaType到`TypeConstrainedMappingJackson2HttpMessageConverter`中，让其能够处理自定义版本头。另外，由于`HalMessageConverterSupportedMediaTypeCustomizer`会添加*application/json*的头兼容JSON请求，需要再其之后添加，否则会被覆盖。

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
