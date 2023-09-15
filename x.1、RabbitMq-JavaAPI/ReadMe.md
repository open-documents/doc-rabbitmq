
Maven依赖：
```xml
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>5.18.0</version>
</dependency>
```


使用原生的API，涉及到的各个组件及每个组件的功能如下：

- ConnectionFactory -- 创建 Connection
- Connection -- 创建 Channel
- Channel -- 完成主要功能，包含：1）交换机、队列的声明与绑定；2）消息的发送；3）消息的消费。