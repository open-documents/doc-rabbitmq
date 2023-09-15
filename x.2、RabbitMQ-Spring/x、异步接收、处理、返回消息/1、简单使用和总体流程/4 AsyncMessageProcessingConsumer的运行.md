

# run

```java
/* --------------------------------- AsyncMessageProcessingConsumer ---------------------------------
 * 描述: 
 *   -  该方法来源于Runnable接口 
 *   -  
 */
public void run() { // NOSONAR - line count  
    if (!isActive()) {  
        this.start.countDown();  
        return;   
    }  
  
    boolean aborted = false;  
  
    this.consumer.setLocallyTransacted(isChannelLocallyTransacted());  
  
    String routingLookupKey = getRoutingLookupKey();  
    if (routingLookupKey != null) {  
        SimpleResourceHolder.bind(getRoutingConnectionFactory(), routingLookupKey); // NOSONAR both never null  
    }  

	// this.consumer
    if (this.consumer.getQueueCount() < 1) {  
      if (logger.isDebugEnabled()) {  
         logger.debug("Consumer stopping; no queues for " + this.consumer);  
      }  
      SimpleMessageListenerContainer.this.cancellationLock.release(this.consumer);  
      if (getApplicationEventPublisher() != null) {  
         getApplicationEventPublisher().publishEvent(  
               new AsyncConsumerStoppedEvent(SimpleMessageListenerContainer.this, this.consumer));  
      }  
      this.start.countDown();  
      return;   }  
  
    try {  
        initialize();  
        while (isActive(this.consumer) || this.consumer.hasDelivery() || !this.consumer.cancelled()) {  
            mainLoop();  
        }  
    }
    catch (InterruptedException e) {  
        ... // log
        Thread.currentThread().interrupt();  
        aborted = true;  
        publishConsumerFailedEvent("Consumer thread interrupted, processing stopped", true, e);  
    }  
    catch (QueuesNotAvailableException ex) {  
        logger.error("Consumer threw missing queues exception, fatal=" + isMissingQueuesFatal(), ex);  
        if (isMissingQueuesFatal()) {  
            this.startupException = ex;  
            // Fatal, but no point re-throwing, so just abort.  
            aborted = true;  
        }  
        publishConsumerFailedEvent("Consumer queue(s) not available", aborted, ex);  
    }  
   catch (FatalListenerStartupException ex) {  
      logger.error("Consumer received fatal exception on startup", ex);  
      this.startupException = ex;  
      // Fatal, but no point re-throwing, so just abort.  
      aborted = true;  
      publishConsumerFailedEvent("Consumer received fatal exception on startup", true, ex);  
   }  
   catch (FatalListenerExecutionException ex) { // NOSONAR exception as flow control  
      logger.error("Consumer received fatal exception during processing", ex);  
      // Fatal, but no point re-throwing, so just abort.  
      aborted = true;  
      publishConsumerFailedEvent("Consumer received fatal exception during processing", true, ex);  
   }  
   catch (PossibleAuthenticationFailureException ex) {  
      logger.error("Consumer received fatal=" + isPossibleAuthenticationFailureFatal() +  
            " exception during processing", ex);  
      if (isPossibleAuthenticationFailureFatal()) {  
         this.startupException =  
               new FatalListenerStartupException("Authentication failure",  
                     new AmqpAuthenticationException(ex));  
         // Fatal, but no point re-throwing, so just abort.  
         aborted = true;  
      }  
      publishConsumerFailedEvent("Consumer received PossibleAuthenticationFailure during startup", aborted, ex);  
   }  
   catch (ShutdownSignalException e) {  
      if (RabbitUtils.isNormalShutdown(e)) {  
         if (logger.isDebugEnabled()) {  
            logger.debug("Consumer received Shutdown Signal, processing stopped: " + e.getMessage());  
         }  
      }  
      else {  
         logConsumerException(e);  
      }  
   }  
   catch (AmqpIOException e) {  
      if (e.getCause() instanceof IOException && e.getCause().getCause() instanceof ShutdownSignalException  
            && e.getCause().getCause().getMessage().contains("in exclusive use")) {  
         getExclusiveConsumerExceptionLogger().log(logger,  
               "Exclusive consumer failure", e.getCause().getCause());  
         publishConsumerFailedEvent("Consumer raised exception, attempting restart", false, e);  
      }  
      else {  
         logConsumerException(e);  
      }  
   }  
   catch (Error e) { //NOSONAR  
      logger.error("Consumer thread error, thread abort.", e);  
      publishConsumerFailedEvent("Consumer threw an Error", true, e);  
      getJavaLangErrorHandler().handle(e);  
      aborted = true;  
   }  
   catch (Throwable t) { //NOSONAR  
      // by now, it must be an exception      if (isActive()) {  
         logConsumerException(t);  
      }  
   }  
   finally {  
      if (getTransactionManager() != null) {  
         ConsumerChannelRegistry.unRegisterConsumerChannel();  
      }  
   }  
  
   // In all cases count down to allow container to progress beyond startup  
   this.start.countDown();  
  
   killOrRestart(aborted);  
  
   if (routingLookupKey != null) {  
      SimpleResourceHolder.unbind(getRoutingConnectionFactory()); // NOSONAR never null here  
   }  
}
```
