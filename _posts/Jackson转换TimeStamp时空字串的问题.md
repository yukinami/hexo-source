title: Jackson转换TimeStamp时空字串的问题
date: 2016-05-21 15:45:59
tags:
- 经验错误
---

## 问题

当使用Jackson进行json数据转换时，如果目标字段类型为TimeStamp并且要转换的值为空字串时会报空指针错误。

## 原因

下面是Timestamp反序列化器的代码

```
public static class TimestampDeserializer extends DateBasedDeserializer<Timestamp>
{
    public TimestampDeserializer() { super(Timestamp.class); }
    public TimestampDeserializer(TimestampDeserializer src, DateFormat df, String formatString) {
        super(src, df, formatString);
    }

    @Override
    protected TimestampDeserializer withDateFormat(DateFormat df, String formatString) {
        return new TimestampDeserializer(this, df, formatString);
    }
    
    @Override
    public java.sql.Timestamp deserialize(JsonParser jp, DeserializationContext ctxt) throws IOException
    {
        return new Timestamp(_parseDate(jp, ctxt).getTime());
    }
}
```

通过`_parseDate`方法转换为Date后，并没有判断是否转换成功而直接调用了getTime方法。

## 解决

添加一个自定义的TimeStamp的Deserializer，然后通过SimpleModule来注册到ObjectMapper中。



[1]: http://www.leveluplunch.com/java/tutorials/033-custom-jackson-date-deserializer/
