
## Consumer优先级

参考文档：
- `https://www.rabbitmq.com/consumer-priority.html`


```java
container.setConsumerArguments(Collections.<String, Object>singletonMap("x-priority", Integer.valueOf(10)));
```

```xml
<rabbit:listener-container connection-factory="rabbitConnectionFactory">   <rabbit:listener queues="some.queue" ref="somePojo" method="handle" priority= "10" /> </rabbit:listener-container>
```
