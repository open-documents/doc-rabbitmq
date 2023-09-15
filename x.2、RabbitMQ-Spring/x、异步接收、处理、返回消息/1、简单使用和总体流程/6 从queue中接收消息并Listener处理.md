
```java
/* --------------------------------------- AsyncMessageProcessingConsumer ---------------------------------------
 */
private void mainLoop() throws Exception { // NOSONAR Exception  
    try {  
        boolean receivedOk = receiveAndExecute(this.consumer); // At least one message received  
        if (SimpleMessageListenerContainer.this.maxConcurrentConsumers != null) {  
            checkAdjust(receivedOk);  
        }  
        long idleEventInterval = getIdleEventInterval();  
        if (idleEventInterval > 0) {  
            if (receivedOk) {  
	            updateLastReceive();  
            }  
            else {  
	            long now = System.currentTimeMillis();  
	            long lastAlertAt = SimpleMessageListenerContainer.this.lastNoMessageAlert.get();  
	            long lastReceive = getLastReceive();  
	            if (now > lastReceive + idleEventInterval  
                    && now > lastAlertAt + idleEventInterval  
                    && SimpleMessageListenerContainer.this.lastNoMessageAlert.compareAndSet(lastAlertAt, now)) {  
	                publishIdleContainerEvent(now - lastReceive);  
                }  
            }  
        }  
    }  
    catch (ListenerExecutionFailedException ex) { ... }  
    catch (AmqpRejectAndDontRequeueException rejectEx) { ... }  
}
```
# 使用事务执行主体逻辑（receiveAndExecute()）

```java
/* -------------------------------- SimpleMessageListenerContainer --------------------------------
 */
private boolean receiveAndExecute(final BlockingQueueConsumer consumer) throws Exception { // NOSONAR  
  
    PlatformTransactionManager transactionManager = getTransactionManager();  
    if (transactionManager != null) {  
        try {  
            if (this.transactionTemplate == null) {  
	            this.transactionTemplate = new TransactionTemplate(
		            transactionManager, 
		            getTransactionAttribute()
		        );  
            }  
            return this.transactionTemplate.execute(
				status -> { // NOSONAR null never returned  
                    RabbitResourceHolder resourceHolder = ConnectionFactoryUtils.bindResourceToTransaction(  
                        new RabbitResourceHolder(
	                        consumer.getChannel(), 
	                        false
	                    ),  
                        getConnectionFactory(), 
                        true);  
                    // unbound in ResourceHolderSynchronization.beforeCompletion()  
                    try {  
                        return doReceiveAndExecute(consumer);  
                    }  
                    catch (RuntimeException e1) {  
                        prepareHolderForRollback(resourceHolder, e1);  
                        throw e1;  
                    }  
                  catch (Exception e2) {  
                     throw new WrappedTransactionException(e2);  
                  }  
               });  
      }  
      catch (WrappedTransactionException e) { // NOSONAR exception flow control  
         throw (Exception) e.getCause();  
      }  
   }  
  
   return doReceiveAndExecute(consumer);  
  
}

```
## 主体逻辑 -- doReceiveAndExecute()
```java
/* ------------------------------ SimpleMessageListenerContainer ------------------------------
 * 参数: 
 *   -  consumer: this.consumer
 */
private boolean doReceiveAndExecute(BlockingQueueConsumer consumer) throws Exception { //NOSONAR  
  
    Channel channel = consumer.getChannel();  
  
    List<Message> messages = null;  
    long deliveryTag = 0;  
  
    for (int i = 0; i < this.batchSize; i++) {  

		// 主要逻辑:
		// 1. 
        Message message = consumer.nextMessage(this.receiveTimeout);  // 方法 1
        if (message == null) {  
            break;  
        }  
        if (this.consumerBatchEnabled) {  
	        // 1. getAfterReceivePostProcessors(): 
	        //   - 父类AbstractMessageListenerContainer方法
	        //   - 返回值 this.afterReceivePostProcessors
	        // 2. this.afterReceivePostProcessors: 类型 Collection<MessagePostProcessor>
            Collection<MessagePostProcessor> afterReceivePostProcessors = getAfterReceivePostProcessors();  
            if (afterReceivePostProcessors != null) {  
	            Message original = message;  
	            deliveryTag = message.getMessageProperties().getDeliveryTag();  
	            for (MessagePostProcessor processor : getAfterReceivePostProcessors()) {  
	                message = processor.postProcessMessage(message);  
		            if (message == null) {  
			            ... // log
		                break;  
                    }  
                }  
            }  
            if (message != null) {  
	            if (messages == null) {  
	                messages = new ArrayList<>(this.batchSize);  
	            }  
	            // 
                if (isDeBatchingEnabled() 
	                && getBatchingStrategy().canDebatch(message.getMessageProperties())) {  
	                final List<Message> messageList = messages;  
	                getBatchingStrategy().deBatch(message, fragment -> messageList.add(fragment));  
	            }  
	            else {  
	                messages.add(message);  
                }  
            }  
        }  
        else {  
	        // 
            messages = debatch(message);  // 方法 2.1
            // 
            if (messages != null) {  
	            break;  
            }  
            try {  
	            executeListener(channel, message);  // 方法 2.2
            }  
            catch (ImmediateAcknowledgeAmqpException e) {  
	            ... // log
                break;  
            }  
            catch (Exception ex) {  
	            if (causeChainHasImmediateAcknowledgeAmqpException(ex)) {  
	                ... // log 
                    break;  
                }  
	            long tagToRollback = isAsyncReplies()  
                    ? message.getMessageProperties().getDeliveryTag()  
                    : -1;  
	            if (getTransactionManager() != null) {  
	                if (getTransactionAttribute().rollbackOn(ex)) {  
	                    RabbitResourceHolder resourceHolder = (RabbitResourceHolder) TransactionSynchronizationManager  
	                        .getResource(getConnectionFactory());  
	                    if (resourceHolder != null) {  
	                        consumer.clearDeliveryTags();  
	                    }  
	                    else {  
                        /*  
                        * If we don't actually have a transaction, we have to roll back                      
                        * manually. See prepareHolderForRollback().                     
                        **/                     
	                        consumer.rollbackOnExceptionIfNecessary(ex, tagToRollback);  
	                    }  
	                    throw ex; // encompassing transaction will handle the rollback.  
	                }  
	                else {  
						... // log
	                    break;  
	                }  
	            }  
	            else {  
	                consumer.rollbackOnExceptionIfNecessary(ex, tagToRollback);  
	                throw ex;  
	            }  
            }  
        }  
    }

    if (messages != null) {  
        executeWithList(channel, messages, deliveryTag, consumer);  // 方法 3
    }  
  
    return consumer.commitIfNecessary(isChannelLocallyTransacted());  
  
}
```
## 方法 1 -- 从queue中获取消息
```java
/* -------------------------------- BlockingQueueConsumer --------------------------------
 * 描述: 
 *   1.  从queue中获取消息
 *   2.  使用this.messagePropertiesConverter对MessageProperties进行处理, 然后设置 MessageProperties的
 *       ConsumerTag 和 ConsumerQueue 
 */
public Message nextMessage(long timeout) throws InterruptedException, ShutdownSignalException {  

	... // log

    checkShutdown();  // 内部方法 1
    if (this.missingQueues.size() > 0) {  
        checkMissingQueues();  // 内部方法 2
    }  
    // this.queue:
    // - 类型 BlockingQueue<Delivery>
    Message message = handle(this.queue.poll(timeout, TimeUnit.MILLISECONDS));  
    if (message == null && this.cancelled.get()) {  
        throw new ConsumerCancelledException();  
    }  
    return message;  
}
// --------------------------------- 内部方法 1 ---------------------------------
private void checkShutdown() {  
    if (this.shutdown != null) {  
        throw Utility.fixStackTrace(this.shutdown);  
    }  
}
// --------------------------------- 内部方法 2 ---------------------------------
private void checkMissingQueues() {  
    long now = System.currentTimeMillis();  
    // 当前时间 已经过了 最后一次重试声明时间所需要的间隔, 因此可以重试声明
    if (now - this.retryDeclarationInterval > this.lastRetryDeclaration) {  
        synchronized (this.missingQueues) {  
            Iterator<String> iterator = this.missingQueues.iterator();  
            while (iterator.hasNext()) {  
	            boolean available = true;  
	            String queueToCheck = iterator.next();  
	            Connection connection = null; // NOSONAR - RabbitUtils  
	            Channel channelForCheck = null;  
	            try {  
	                connection = this.connectionFactory.createConnection();  
	                channelForCheck = connection.createChannel(false);  
	                channelForCheck.queueDeclarePassive(queueToCheck);  
	                if (logger.isInfoEnabled()) {  
                     logger.info("Queue '" + queueToCheck + "' is now available");  
	                }  
                }  
	            catch (IOException e) {  
	                available = false;  
	                if (logger.isWarnEnabled()) {  
	                    logger.warn("Queue '" + queueToCheck + "' is still not available");  
                    }  
                }  
	            finally {  
	                RabbitUtils.closeChannel(channelForCheck);  
	                RabbitUtils.closeConnection(connection);  
	            }  
	            if (available) {  
	                try {  
	                    this.consumeFromQueue(queueToCheck);  
	                    iterator.remove();  
	                }  
	                catch (IOException e) {  
	                    throw RabbitExceptionTranslator.convertRabbitAccessException(e);  
                    }  
                }  
            }   
        }  
        this.lastRetryDeclaration = now;  
    }  
}
/* -------------------------------- BlockingQueueConsumer -------------------------------- */
private Message handle(@Nullable Delivery delivery) {  
    if ((delivery == null && this.shutdown != null)) {  
        throw this.shutdown;  
    }  
    if (delivery == null) {  
        return null;  
    }  
    byte[] body = delivery.getBody();  
    Envelope envelope = delivery.getEnvelope();  

	// 对MessageProperties部分进行处理
    MessageProperties messageProperties = this.messagePropertiesConverter.toMessageProperties(  
         delivery.getProperties(), 
         envelope, 
         "UTF-8"
    );  
    messageProperties.setConsumerTag(delivery.getConsumerTag());  
    messageProperties.setConsumerQueue(delivery.getQueue());  

	// 
    Message message = new Message(body, messageProperties);  

	... // log: 从共享消息队列中获取的消息是 message
    this.deliveryTags.add(messageProperties.getDeliveryTag());  
    if (this.transactional && !this.locallyTransacted) {  
        ConnectionFactoryUtils.registerDeliveryTag(
	        this.connectionFactory, 
	        this.channel,  
            delivery.getEnvelope().getDeliveryTag()
        );  
   }  
   return message;  
}
```
## 方法 2.1 -- 
```java
/* ------------------------------ AbstractMessageListenerContainer ------------------------------
 * 描述:
 *   -  该类是 SimpleMessageListenerContainer 的父类
 */
protected List<Message> debatch(Message message) {  
	
    if (this.isBatchListener 
	    && isDeBatchingEnabled()  
        && getBatchingStrategy().canDebatch(message.getMessageProperties())) {  
        final List<Message> messageList = new ArrayList<>();  
        getBatchingStrategy().deBatch(message, fragment -> messageList.add(fragment));  
        return messageList;  
    }  
    return null;  
}
```

## 方法 2.2 -- executeListener()
```java
/* ------------------------------ AbstractMessageListenerContainer ------------------------------
 * 描述: 
 *   -  消息发送到MessageListener
 *   -  执行业务逻辑
 */
protected void executeListener(Channel channel, Object data) {  
    Observation observation;  
    ObservationRegistry registry = getObservationRegistry();  
    if (data instanceof Message message) {  
        observation = RabbitListenerObservation.LISTENER_OBSERVATION.observation(
	        this.observationConvention,  
            DefaultRabbitListenerObservationConvention.INSTANCE,
            () -> new RabbitMessageReceiverContext(message, getListenerId()), registry);  
        observation.observe(() -> executeListenerAndHandleException(channel, data));  
    }  
    else {  
        executeListenerAndHandleException(channel, data);  
    }  
}
/* ------------------------------ AbstractMessageListenerContainer ------------------------------ */
 protected void executeListenerAndHandleException(Channel channel, Object data) {  
    if (!isRunning()) {  
		... // log
        throw new MessageRejectedWhileStoppingException();  
    }  

    Object sample = null;  
    MicrometerHolder micrometerHolder = getMicrometerHolder();  
    if (micrometerHolder != null) {  
        sample = micrometerHolder.start();  
    }  
    try {  
        doExecuteListener(channel, data);  
        if (sample != null) {  
            micrometerHolder.success(
	            sample, 
	            data instanceof Message message 
		            ? message.getMessageProperties().getConsumerQueue()  
	                : queuesAsListString());  
        }  
    }  
   catch (RuntimeException ex) {  
      if (sample != null) {  
         micrometerHolder.failure(sample, data instanceof Message message  
               ? message.getMessageProperties().getConsumerQueue()  
               : queuesAsListString(), ex.getClass().getSimpleName());  
      }  
      Message message;  
      if (data instanceof Message msg) {  
         message = msg;  
      }  
      else {  
         message = ((List<Message>) data).get(0);  
      }  
      checkStatefulRetry(ex, message);  
      handleListenerException(ex);  
      throw ex;  
   }  
}

/* ------------------------------ AbstractMessageListenerContainer ------------------------------ */
private void doExecuteListener(Channel channel, Object data) {  
	// 
    if (data instanceof Message message) {  
	    // this.afterReceivePostProcessors:
	    // - 类型 Collection<MessagePostProcessor>
        if (this.afterReceivePostProcessors != null) {  
            for (MessagePostProcessor processor : this.afterReceivePostProcessors) {  
	            message = processor.postProcessMessage(message);  
	            if (message == null) {  
	                throw new ImmediateAcknowledgeAmqpException("Message Post Processor returned 'null', discarding message");  
                }  
            }  
        }  
        if (this.deBatchingEnabled && this.batchingStrategy.canDebatch(message.getMessageProperties())) {  
            this.batchingStrategy.deBatch(message, fragment -> invokeListener(channel, fragment));  
        }  
        else {  
            invokeListener(channel, message);  
        }  
    }  
    else {  
        invokeListener(channel, data);  
    }  
}

/* ------------------------------ AbstractMessageListenerContainer ------------------------------ */
protected void invokeListener(Channel channel, Object data) {  
	// 
    this.proxy.invokeListener(channel, data);  
}
/* ------------------------------ AbstractMessageListenerContainer ------------------------------ */
protected void actualInvokeListener(Channel channel, Object data) {  
	// 1. getMessageListener(): return this.messageListener;
	// 2. this.messageListener: 类型 MessageListener
    Object listener = getMessageListener();  
    if (listener instanceof ChannelAwareMessageListener chaml) {  
        doInvokeListener(chaml, channel, data);  // 重载方法 1
    }  
    else if (listener instanceof MessageListener msgListener) { // NOSONAR  
        boolean bindChannel = isExposeListenerChannel() && isChannelLocallyTransacted();  
        if (bindChannel) {  
            RabbitResourceHolder resourceHolder = new RabbitResourceHolder(channel, false);  
            resourceHolder.setSynchronizedWithTransaction(true);  
            TransactionSynchronizationManager.bindResource(
	            this.getConnectionFactory(),  
                resourceHolder
            );  
        }  
        try {  
            doInvokeListener(msgListener, data);  // 重载方法 2
        }  
        finally {  
            if (bindChannel) {  
            // unbind if we bound  
	            TransactionSynchronizationManager.unbindResource(this.getConnectionFactory());  
            }  
        }  
    }  
    else if (listener != null) {  
        throw new FatalListenerExecutionException("Only MessageListener and SessionAwareMessageListener supported: "  
            + listener);  
    }  
    else {  
        throw new FatalListenerExecutionException("No message listener specified - see property 'messageListener'");  
    }  
}

/* ------------------------------ AbstractMessageListenerContainer ------------------------------ 
 * 描述: 重载方法1, 该方法有另外一个重载方法(重载方法2)
 */
protected void doInvokeListener(ChannelAwareMessageListener listener, Channel channel, Object data) {  
  
    Message message = null;  
    RabbitResourceHolder resourceHolder = null;  
    Channel channelToUse = channel;  
    boolean boundHere = false;  
    try {  
        if (!isExposeListenerChannel()) {  
            // We need to expose a separate Channel.  
            resourceHolder = getTransactionalResourceHolder();  
            channelToUse = resourceHolder.getChannel();  
            /*  
             * If there is a real transaction, the resource will have been bound; otherwise          
             * we need to bind it temporarily here. Any work done on this channel          
             * will be committed in the finally block.          
             * */         
            if (isChannelLocallyTransacted() 
	            && !TransactionSynchronizationManager.isActualTransactionActive()) {  
	            resourceHolder.setSynchronizedWithTransaction(true);  
	            TransactionSynchronizationManager.bindResource(this.getConnectionFactory(), resourceHolder);  
	            boundHere = true;  
            }  
        }  
	    else {  
            // if locally transacted, bind the current channel to make it available to RabbitTemplate  
            if (isChannelLocallyTransacted()) {  
	            RabbitResourceHolder localResourceHolder = new RabbitResourceHolder(channelToUse, false);  
	            localResourceHolder.setSynchronizedWithTransaction(true);  
                TransactionSynchronizationManager.bindResource(this.getConnectionFactory(), localResourceHolder);  
	            boundHere = true;  
            }  
        }  
        // Actually invoke the message listener...  
        try {  
            if (data instanceof List) {  
	            listener.onMessageBatch((List<Message>) data, channelToUse);  
            }
            else {  
	            message = (Message) data;  
	            listener.onMessage(message, channelToUse);  
            }  
        }
        catch (Exception e) {  
            throw wrapToListenerExecutionFailedExceptionIfNeeded(e, data);  
        }  
    }  
    finally {  
        cleanUpAfterInvoke(resourceHolder, channelToUse, boundHere); // 收尾工作 
    }  
}

/* ------------------------------ AbstractMessageListenerContainer ------------------------------ 
 * 描述: 重载方法2, 该方法有另外一个重载方法(重载方法1)
 */
protected void doInvokeListener(MessageListener listener, Object data) {  
    Message message = null;  
    try {  
        if (data instanceof List) {  
            listener.onMessageBatch((List<Message>) data);  
        }  
        else {  
            message = (Message) data;  
            listener.onMessage(message);  
        }  
    }  
    catch (Exception e) {  
        throw wrapToListenerExecutionFailedExceptionIfNeeded(e, data);  
    }  
}


```

## 方法 3 -- executeWithList()
```java
/* -------------------------------- SimpleMessageListenerContainer --------------------------------
 */
private void executeWithList(Channel channel, List<Message> messages, long deliveryTag,  
      BlockingQueueConsumer consumer) {  
   
    try {  
        executeListener(channel, messages);   // 方法 2.2 
    }  
    catch (ImmediateAcknowledgeAmqpException e) {  
      if (this.logger.isDebugEnabled()) {  
         this.logger.debug("User requested ack for failed delivery '"  
               + e.getMessage() + "' (last in batch): "  
               + deliveryTag);  
      }  
      return;  
   }  
   catch (Exception ex) {  
      if (causeChainHasImmediateAcknowledgeAmqpException(ex)) {  
         if (this.logger.isDebugEnabled()) {  
            this.logger.debug("User requested ack for failed delivery (last in batch): "  
                  + deliveryTag);  
         }  
         return;  
      }  
      if (getTransactionManager() != null) {  
         if (getTransactionAttribute().rollbackOn(ex)) {  
            RabbitResourceHolder resourceHolder = (RabbitResourceHolder) TransactionSynchronizationManager  
                  .getResource(getConnectionFactory());  
            if (resourceHolder != null) {  
               consumer.clearDeliveryTags();  
            }  
            else {  
               /*  
                * If we don't actually have a transaction, we have to roll back                * manually. See prepareHolderForRollback().                */               consumer.rollbackOnExceptionIfNecessary(ex);  
            }  
            throw ex; // encompassing transaction will handle the rollback.  
         }  
         else {  
            if (this.logger.isDebugEnabled()) {  
               this.logger.debug("No rollback for " + ex);  
            }  
         }  
      }  
      else {  
         consumer.rollbackOnExceptionIfNecessary(ex);  
         throw ex;  
      }  
   }  
}
```