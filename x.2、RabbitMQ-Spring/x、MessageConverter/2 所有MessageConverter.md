
参考文档：P99

	先列出一些，其他部分有时间再进行补充。

# 1 SimpleMessageConverter

默认情况下，RabbitTemplate中涉及到的对Message的类型处理都是使用的SimpleMessageConverter。

## Message 到 其他

1）如果入站的Message的 `Content Type` 是"text"开头（如`"text/plain"`）

SimpleMessageConverter会检查content-encoding属性去决定对Message的body转换为String时使用的字符集。默认的字符集是 `UTF-8`，可以通过向RabbitTemplate注入一个设置了`defaultCharset`的SimpleMessageConverter实例来改变。

2）如果入站的Message的 `Content Type` 是 `"application/x-java-serialized-object"`

SimpleMessageConverter会将Message的body部分反序列化为Java对象。

这是不推荐的：producer 和 consumer 不一定都使用Java。

3）如果入站的Message的 `Content Type` 是 其他类型

SimpleMessageConverter会将Message的body部分按原样的 `byte[]` 返回。

## 其他 到 Message

When converting to a Message from an arbitrary Java Object, the SimpleMessageConverter likewise deals with byte arrays, strings, and serializable instances. It converts each of these to bytes (in the case of byte arrays, there is nothing to convert), and it sets the content-type property accordingly. If the Object to be converted does not match one of those types, the Message body is null.

# 2 SerializerMessageConverter

与SimpleMessageConverter基本相同，除了可以配置其他的Spring框架的Serializer和Deserializer实现。

# 3 Jackson2JsonMessageConverter

1）正如前面所提到的，Producer 和 Consumer 可能使用不同的语言，但是Json数据格式是在不同语言间比较通用的。
2）可以向RabbitTemplate实例中注入Jackson2JsonMessageConverter实例来覆盖掉默认的SimpleMessageConverter。
3）使用了 `com.fasterxml.jackson 2.x` 库。

## 配置

```xml
<bean class="org.springframework.amqp.rabbit.core.RabbitTemplate">
	<property name="connectionFactory" ref="rabbitConnectionFactory"/>
	<property name="messageConverter"> 
		<bean class= "org.springframework.amqp.support.converter.Jackson2JsonMessageConverter">   
			<!-- if necessary, override the DefaultClassMapper --> 
			<property name="classMapper" ref="customClassMapper"/>
		</bean> 
	</property>
</bean>
```
如果入站Message不包含
```xml
<bean id="jsonConverterWithDefaultType"   class="o.s.amqp.support.converter.Jackson2JsonMessageConverter">   <property name="classMapper">   <bean class=" org.springframework.amqp.support.converter.DefaultClassMapper">   <property name="defaultType" value="thing1.PurchaseOrder"/>   </bean>   </property> </bean>
```

## ClassMapper

Jackson2JsonMessageConverter从MessageProperties中提取或
使用ClassMapper去

## Message 到 

## 其他 到 Message

