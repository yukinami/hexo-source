title: Spring Annotation-driven Formatting
date: 2016-02-14 16:45:22
tags:
- Spring Framework
---

应用中可能设计很多不同格式的时间、数字的转换和显示。可以通过基于Annotation-driven Formatting来实现。 使用@NumberFormat来格式化java.lang.Number字段，使用@DateTimeFormat来格式化java.util.Date, java.util.Calendar, java.util.Long, 或者Joda Time字段。

## AnnotationFormatterFactory

为了使用注解格式化，需要注册`AnnotationFormatterFactory`的实例，下面是个例子

<!--more-->

```
public final class NumberFormatAnnotationFormatterFactory
        implements AnnotationFormatterFactory<NumberFormat> {

    public Set<Class<?>> getFieldTypes() {
        return new HashSet<Class<?>>(asList(new Class<?>[] {
            Short.class, Integer.class, Long.class, Float.class,
            Double.class, BigDecimal.class, BigInteger.class }));
    }

    public Printer<Number> getPrinter(NumberFormat annotation, Class<?> fieldType) {
        return configureFormatterFrom(annotation, fieldType);
    }

    public Parser<Number> getParser(NumberFormat annotation, Class<?> fieldType) {
        return configureFormatterFrom(annotation, fieldType);
    }

    private Formatter<Number> configureFormatterFrom(NumberFormat annotation,
            Class<?> fieldType) {
        if (!annotation.pattern().isEmpty()) {
            return new NumberFormatter(annotation.pattern());
        } else {
            Style style = annotation.style();
            if (style == Style.PERCENT) {
                return new PercentFormatter();
            } else if (style == Style.CURRENCY) {
                return new CurrencyFormatter();
            } else {
                return new NumberFormatter();
            }
        }
    }
}
```

Spring提供了很多默认的实现

- DateTimeFormatAnnotationFormatterFactory 支持Date，Calendar，Long
- NumberFormatAnnotationFormatterFactory 支持Byte，Short，Integer，Long，BigInteger，Float，Double，BigDecimal

还有`JodaDateTimeFormatAnnotationFormatterFactory`, `Jsr310DateTimeFormatAnnotationFormatterFactory`, `Jsr354NumberFormatAnnotationFormatterFactory`

## 注册AnnotationFormatterFactory

`FormatterRegistrar`接口是一个注册formatters和converters的SPI。Spring提供了`DateTimeFormatterRegistrar`，`DateTimeFormatterRegistrar`，`JodaTimeFormatterRegistrar`来注册对应的AnnotationFormatterFactory。

另外这几个Registrar会在`DefaultFormattingConversionService`中根据classpath中包含的类自动的进行注册

```
/**
 * Add formatters appropriate for most environments: including number formatters,
 * JSR-354 Money & Currency formatters, JSR-310 Date-Time and/or Joda-Time formatters,
 * depending on the presence of the corresponding API on the classpath.
 * @param formatterRegistry the service to register default formatters with
 */
public static void addDefaultFormatters(FormatterRegistry formatterRegistry) {
	// Default handling of number values
	formatterRegistry.addFormatterForFieldAnnotation(new NumberFormatAnnotationFormatterFactory());

	// Default handling of monetary values
	if (jsr354Present) {
		formatterRegistry.addFormatter(new CurrencyUnitFormatter());
		formatterRegistry.addFormatter(new MonetaryAmountFormatter());
		formatterRegistry.addFormatterForFieldAnnotation(new Jsr354NumberFormatAnnotationFormatterFactory());
	}

	// Default handling of date-time values
	if (jsr310Present) {
		// just handling JSR-310 specific date and time types
		new DateTimeFormatterRegistrar().registerFormatters(formatterRegistry);
	}
	if (jodaTimePresent) {
		// handles Joda-specific types as well as Date, Calendar, Long
		new JodaTimeFormatterRegistrar().registerFormatters(formatterRegistry);
	}
	else {
		// regular DateFormat-based Date, Calendar, Long converters
		new DateFormatterRegistrar().registerFormatters(formatterRegistry);
	}
	}
```

NOTE: DefaultFormattingConversionService由FormattingConversionServiceFactoryBean注册

## 配置全局的date & time格式

默认情况下，没有@DateTimeFormat注解的日期和时间，会使用`DateFormat.SHORT`格式来转换。 可以自定义这个全局的格式。

首先需要确保Spring不自动注册默认的formatters，然后自己手动注册。以DateFormatterRegistrar为例

```
@Configuration
public class AppConfig {

    @Bean
    public FormattingConversionService conversionService() {

        // Use the DefaultFormattingConversionService but do not register defaults
        DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService(false);

        // Ensure @NumberFormat is still supported
        conversionService.addFormatterForFieldAnnotation(new NumberFormatAnnotationFormatterFactory());

        // Register date conversion with a specific global format
        DateFormatterRegistrar registrar = new DateFormatterRegistrar();
        registrar.setFormatter(new DateFormatter("yyyyMMdd"));
        registrar.registerFormatters(conversionService);

        return conversionService;
    }
}
```

### 一些问题

上面其实就是额外再注册了一个DateFormatter，但是由于某些原因，DateFormatter必须在DateTimeFormatAnnotationFormatterFactory之前注册，否则@DateTimeFormat注解将会失效。 

大概原因是`GenericConversionService`在查找对应的Converter时的策略问题

ConvertersForPair

```
private static class ConvertersForPair {

	private final LinkedList<GenericConverter> converters = new LinkedList<GenericConverter>();

	public void add(GenericConverter converter) {
		this.converters.addFirst(converter);
	}

	public GenericConverter getConverter(TypeDescriptor sourceType, TypeDescriptor targetType) {
		for (GenericConverter converter : this.converters) {
			if (!(converter instanceof ConditionalGenericConverter) ||
					((ConditionalGenericConverter) converter).matches(sourceType, targetType)) {
				return converter;
			}
		}
		return null;
	}

	@Override
	public String toString() {
		return StringUtils.collectionToCommaDelimitedString(this.converters);
	}
}
```

这里如果converter不是ConditionalGenericConverter的实例，那么是直接通过的，如果DateFormatter的顺序在前，便会直接返回DateFormatter的converter，而不考虑字段上面的注解。

由于ConvertersForPair中add方法是添加到converters的头部的，所以DateFormatter的注册顺序得在前面。但是配合Spring Boot的WebMvcAutoConfigurationAdapter注册全局的日期格式时，由于它初始化的顺序永远在自定义的WebMvcConfigurerAdapter之后，所以只能不使用`spring.mvc.date-format`的设置，自己手动添加DateFormatter。

````
public void addFormatters(FormatterRegistry registry) {
    super.addFormatters(registry);
    registry.addFormatter(new DateFormatter("yyyy-MM-dd HH:mm"));
    registry.addFormatterForFieldAnnotation(new DateTimeFormatAnnotationFormatterFactory());
}
```
