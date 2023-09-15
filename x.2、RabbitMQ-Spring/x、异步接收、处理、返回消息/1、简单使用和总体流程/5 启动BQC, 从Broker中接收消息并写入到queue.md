
# initialize()
```java
/* --------------------------------- AsyncMessageProcessingConsumer ---------------------------------
 */
private void initialize() throws Throwable { // NOSONAR  
    try {  

        redeclareElementsIfNecessary();  

		// 
        this.consumer.start();  // 逻辑方法
        this.start.countDown();  
    }  
    catch (QueuesNotAvailableException e) {  
        if (isMissingQueuesFatal()) {  
            throw e;  
        }  
        else {  
            this.start.countDown();  
            handleStartupFailure(this.consumer.getBackOffExecution());  
            throw e;  
        }  
    }  
   catch (FatalListenerStartupException ex) {  
      if (isPossibleAuthenticationFailureFatal()) {  
         throw ex;  
      }  
      else {  
         Throwable possibleAuthException = ex.getCause().getCause();  
         if (!(possibleAuthException instanceof PossibleAuthenticationFailureException)) {  
            throw ex;  
         }  
         else {  
            this.start.countDown();  
            handleStartupFailure(this.consumer.getBackOffExecution());  
            throw possibleAuthException;  
         }  
      }  
   }  
   catch (Throwable t) { //NOSONAR  
      this.start.countDown();  
      handleStartupFailure(this.consumer.getBackOffExecution());  
      throw t;  
   }  
  
   if (getTransactionManager() != null) {  
      /*  
       * Register the consumer's channel so it will be used by the transaction manager       * if it's an instance of RabbitTransactionManager.       */      
       ConsumerChannelRegistry.registerConsumerChannel(this.consumer.getChannel(), getConnectionFactory());  
   }  
}
```


# 逻辑方法 -- start()
```java
/* ------------------------------------ BlockingQueueConsumer ------------------------------------
 */ 
public void start() throws AmqpException {  

	... // log
  
    this.thread = Thread.currentThread();  
  
    try {  
        this.resourceHolder = ConnectionFactoryUtils.getTransactionalResourceHolder(
	        this.connectionFactory,  
            this.transactional
        );  
        this.channel = this.resourceHolder.getChannel();  
        ClosingRecoveryListener.addRecoveryListenerIfNecessary(this.channel); // NOSONAR never null here  
    }  
    catch (AmqpAuthenticationException e) {  
        throw new FatalListenerStartupException("Authentication failure", e);  
    }  
    this.deliveryTags.clear();  
    this.activeObjectCounter.add(this);  
  
    passiveDeclarations();       // 方法 1
    setQosAndCreateConsumers();  // 方法 2
}
```

## 方法 2 -- 设置服务项 & 创建InternalConsumer从Broker接收消息 
```java
private void setQosAndCreateConsumers() {  
    if (this.consumeDelay > 0) {  
        try {  
            Thread.sleep(this.consumeDelay);  
        }  
        catch (InterruptedException e) {  
            Thread.currentThread().interrupt();  
        }  
    }  
    if (!this.acknowledgeMode.isAutoAck() && !cancelled()) {  
        try {  
            this.channel.basicQos(this.prefetchCount, this.globalQos);  
        }  
        catch (IOException e) {  
            this.activeObjectCounter.release(this);  
            throw new AmqpIOException(e);  
        }  
    }  
  
    try {  
        if (!cancelled()) {  
            for (String queueName : this.queues) {  
                if (!this.missingQueues.contains(queueName)) {  
                    consumeFromQueue(queueName);  
                }  
            }  
        }  
    }  
    catch (IOException e) {  
        throw RabbitExceptionTranslator.convertRabbitAccessException(e);  
    }  
}

private void consumeFromQueue(String queue) throws IOException {  
    InternalConsumer consumer = new InternalConsumer(this.channel, queue);  
    String consumerTag = this.channel.basicConsume(
	    queue, 
        this.acknowledgeMode.isAutoAck(),  
        (this.tagStrategy != null ? this.tagStrategy.createConsumerTag(queue) : ""), 
        this.noLocal,  
        this.exclusive, 
        this.consumerArgs,  
        consumer);
  
    if (consumerTag != null) {  
        this.consumers.put(queue, consumer);  
		... // log
    }  
    else {  
        ... // log
    }  
}
```



