
# 开启注解声明消息监听器

```java
@Configuration 
@EnableRabbit public class AppConfig {   @Bean   public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory() {   SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();   factory.setConnectionFactory(connectionFactory());   factory.setConcurrentConsumers(3);   factory.setMaxConcurrentConsumers(10);   factory.setContainerCustomizer(container -> /* customize the container */ );   return factory;  } }
```
