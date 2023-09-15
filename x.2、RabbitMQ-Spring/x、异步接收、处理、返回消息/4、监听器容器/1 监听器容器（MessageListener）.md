
监听器容器特点：
- MessageListener Container实现了LifeCycle接口。
- 必须提供ConnectionFactory引用。
- 必须监听器监听的queue名称或Queue实例。





# 队列自动删除与重新声明




```java
<rabbit:queue id="otherAnon" declared-by="containerAdmin" /> <rabbit:direct-exchange name="otherExchange" auto-delete="true" declared-by= "containerAdmin">   <rabbit:bindings>   <rabbit:binding queue="otherAnon" key="otherAnon" />   </rabbit:bindings> </rabbit:direct-exchange> <rabbit:listener-container id="container2" auto-startup="false">   <rabbit:listener id="listener2" ref="foo" queues="otherAnon" admin= "containerAdmin" /> </rabbit:listener-container> <rabbit:admin id="containerAdmin" connection-factory="rabbitConnectionFactory"   auto-startup="false" />
```