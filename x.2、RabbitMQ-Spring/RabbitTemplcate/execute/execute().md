
# 一、execute()
```java
public <T> T execute(ChannelCallback<T> action) {  
    return execute(action, getConnectionFactory());  
}
private <T> T execute(final ChannelCallback<T> action, final ConnectionFactory connectionFactory) {  
    if (this.retryTemplate != null) {  
        try {  
            return this.retryTemplate.execute(  
               (RetryCallback<T, Exception>) context -> doExecute(action, connectionFactory),  
               (RecoveryCallback<T>) this.recoveryCallback
            );  
        }  
		... // catch
		... // catch
    }  
    else {  
        return doExecute(action, connectionFactory);  
    }  
}
```

# 二、doExecute()
```java
/* -------------------------------------- RabbitTemplate --------------------------------------
 */
private <T> T doExecute(ChannelCallback<T> action, ConnectionFactory connectionFactory) { 

	// NOSONAR complexity  
	... // assert

    Channel channel = null;  
    boolean invokeScope = false;  

    // 1. this.activeTemplateCallbacks:
    //  - 类型       -- AtomicInteger
    //  - 默认(初)值 -- new AtomicInteger()
    if (this.activeTemplateCallbacks.get() > 0) {  
		// 1. this.dedicatedChannels:
	    //  - 类型       -- ThreadLocal<Channel>
	    //  - 默认(初)值 -- new ThreadLocal<Channel>()
        channel = this.dedicatedChannels.get();  
    }
    RabbitResourceHolder resourceHolder = null;  
    Connection connection = null; // NOSONAR (close)  
    if (channel == null) {  
	    // isChannelTransacted():
	    // - 返回this.transactional
	    // - this.transactional类型为boolean
        if (isChannelTransacted()) {  
            resourceHolder = ConnectionFactoryUtils.getTransactionalResourceHolder(
	            connectionFactory,  
                true, 
                this.usePublisherConnection
            );  
            channel = resourceHolder.getChannel();  
            if (channel == null) {  
	            ConnectionFactoryUtils.releaseResources(resourceHolder);  
	            throw new IllegalStateException("Resource holder returned a null channel");  
            }  
        }  
        else {  
            connection = ConnectionFactoryUtils.createConnection(
	            connectionFactory,  
                this.usePublisherConnection
            ); // NOSONAR - RabbitUtils closes  
            if (connection == null) {  
	            throw new IllegalStateException("Connection factory returned a null connection");  
            }  
            try {  
	            channel = connection.createChannel(false);  
	            if (channel == null) {
	                throw new IllegalStateException("Connection returned a null channel");  
	            }  
            }  
            catch (RuntimeException e) {  
	            RabbitUtils.closeConnection(connection);  
	            throw e;  
            }  
        }  
    }  
    else {  
        invokeScope = true;  
    }  
    try {
        return invokeAction(action, connectionFactory, channel);  
    }  
    catch (Exception ex) {  
        if (isChannelLocallyTransacted(channel)) {  
            resourceHolder.rollbackAll();  
        }  
        throw convertRabbitAccessException(ex);  
    }  
    finally {  
        cleanUpAfterAction(channel, invokeScope, resourceHolder, connection);  
    }  
}
```
## 1 invokeAction()
```java
private <T> T invokeAction(ChannelCallback<T> action, ConnectionFactory connectionFactory, Channel channel) throws Exception { // NOSONAR see the callback  

	
    if (isPublisherConfirmsOrReturns(connectionFactory)) {  
        addListener(channel);
    }  
  
    return action.doInRabbit(channel);  
}
```



# 创建连接

```java
/* -------------------------------------- ConnectionFactoryUtils --------------------------------------
 */
public static Connection createConnection(final ConnectionFactory connectionFactory,  
      final boolean publisherConnectionIfPossible) {  
  
   if (publisherConnectionIfPossible) {  
      ConnectionFactory publisherFactory = connectionFactory.getPublisherConnectionFactory();  
      if (publisherFactory != null) {  
         return publisherFactory.createConnection();  
      }  
   }  
   return connectionFactory.createConnection();  
}
```