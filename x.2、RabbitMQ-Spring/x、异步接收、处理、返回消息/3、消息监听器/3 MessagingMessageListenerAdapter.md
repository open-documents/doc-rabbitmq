


# 一、接收并处理消息 -- onMessage()
```java
/* -------------------------------------- MessagingMessageListenerAdapter --------------------------------------
 */
public void onMessage(org.springframework.amqp.core.Message amqpMessage, Channel channel) throws Exception { // NOSONAR  
    Message<?> message = null;  
    try {  
        message = toMessagingMessage(amqpMessage);                             // 1
        invokeHandlerAndProcessResult(amqpMessage, channel, message);          // 2
    }  
    catch (ListenerExecutionFailedException ex) {  
        handleException(amqpMessage, channel, message, ex);  
    }  
    catch (ReplyFailureException ex) { // NOSONAR  
        throw ex;  
    }  
    catch (Exception ex) { // NOSONAR  
        handleException(amqpMessage, channel, message, new ListenerExecutionFailedException(  
            "Failed to convert message", ex, amq pMessage));  
    }  
}
```

# 1 toMessagingMessage()
```java
/* -------------------------------------- MessagingMessageListenerAdapter --------------------------------------
 */
protected Message<?> toMessagingMessage(org.springframework.amqp.core.Message amqpMessage) {  
    return (Message<?>) getMessagingMessageConverter().fromMessage(amqpMessage);  
}
```

# 2 invokeHandlerAndProcessResult()
```java
/* -------------------------------------- MessagingMessageListenerAdapter --------------------------------------
 */
protected void invokeHandlerAndProcessResult(@Nullable org.springframework.amqp.core.Message amqpMessage, Channel channel, Message<?> message) throws Exception { // NOSONAR  
  
    boolean projectionUsed = amqpMessage == null ? false : amqpMessage.getMessageProperties().isProjectionUsed();  
    if (projectionUsed) {  
        amqpMessage.getMessageProperties().setProjectionUsed(false);  
    }  
	... // log
    InvocationResult result = null;  
    if (this.messagingMessageConverter.method == null && amqpMessage != null) {  
        amqpMessage.getMessageProperties().setTargetMethod(this.handlerAdapter.getMethodFor(message.getPayload()));  
    }  
    result = invokeHandler(amqpMessage, channel, message);  
    if (result.getReturnValue() != null) {  
        handleResult(result, amqpMessage, channel, message);  
    }  
    else {... // log}  
}
```
## 1）invokeHandler
```java
/* -------------------------------------- MessagingMessageListenerAdapter --------------------------------------
 */
private InvocationResult invokeHandler(@Nullable org.springframework.amqp.core.Message amqpMessage, Channel channel, Message<?> message) {  
    try {  
        if (amqpMessage == null) {  
            return this.handlerAdapter.invoke(message, channel);  
        }  
        else {  
            return this.handlerAdapter.invoke(message, amqpMessage, channel, amqpMessage.getMessageProperties());  
        }  
    }  
   catch (MessagingException ex) {  
      throw new ListenerExecutionFailedException(createMessagingErrorMessage("Listener method could not " +  
            "be invoked with the incoming message", message.getPayload()), ex, amqpMessage);  
   }  
   catch (Exception ex) {  
      throw new ListenerExecutionFailedException("Listener method '" +  
            this.handlerAdapter.getMethodAsString(message.getPayload()) + "' threw exception", ex, amqpMessage);  
   }  
}
/* -------------------------------------- HandlerAdapter --------------------------------------
 */

public InvocationResult invoke(@Nullable Message<?> message, Object... providedArgs) throws Exception { // NOSONAR  

	// 1. this.invokerHandlerMethod: 
	// 类型: InvocableHandlerMethod
    if (this.invokerHandlerMethod != null) { // NOSONAR (nullable message)  
        return new InvocationResult(
	        this.invokerHandlerMethod.invoke(message, providedArgs),  
            null, 
            this.invokerHandlerMethod.getMethod().getGenericReturnType(),  
            this.invokerHandlerMethod.getBean(),  
            this.invokerHandlerMethod.getMethod()
        );  
    }  
    else if (this.delegatingHandler.hasDefaultHandler()) {  
        // Needed to avoid returning raw Message which matches Object  
        Object[] args = new Object[providedArgs.length + 1];  
        args[0] = message.getPayload();  
        System.arraycopy(providedArgs, 0, args, 1, providedArgs.length);  
        return this.delegatingHandler.invoke(message, args);  
    }  
    else {  
        return this.delegatingHandler.invoke(message, providedArgs);  
    }  
}
```

## 2种调用监听器方法的方式

```java
/* ----------------------------------------- InvocableHandlerMethod -----------------------------------------
 */
public Object invoke(Message<?> message, Object... providedArgs) throws Exception {  
   Object[] args = getMethodArgumentValues(message, providedArgs);  
   if (logger.isTraceEnabled()) {  
      logger.trace("Arguments: " + Arrays.toString(args));  
   }  
   return doInvoke(args);  
}
```

```java
public InvocationResult invoke(Message<?> message, Object... providedArgs) throws Exception { // NOSONAR  
   Class<? extends Object> payloadClass = message.getPayload().getClass();  
   InvocableHandlerMethod handler = getHandlerForPayload(payloadClass);  
   if (this.validator != null && this.defaultHandler != null) {  
      MethodParameter parameter = this.payloadMethodParameters.get(handler);  
      if (parameter != null && this.validator.supportsParameter(parameter)) {  
         this.validator.validate(message, parameter, message.getPayload());  
      }  
   }  
   Object result = handler.invoke(message, providedArgs);  
   if (message.getHeaders().get(AmqpHeaders.REPLY_TO) == null) {  
      Expression replyTo = this.handlerSendTo.get(handler);  
      if (replyTo != null) {  
         return new InvocationResult(result, replyTo, handler.getMethod().getGenericReturnType(),  
               handler.getBean(), handler.getMethod());  
      }  
   }  
   return new InvocationResult(result, null, handler.getMethod().getGenericReturnType(), handler.getBean(),  
         handler.getMethod());  
}
```

## 2）

```java
/* ---------------------------------------- AbstractAdaptableMessageListener ----------------------------------------
 */
protected void handleResult(InvocationResult resultArg, Message request, Channel channel, Object source) {  
   if (channel != null) {  
      if (resultArg.getReturnValue() instanceof CompletableFuture<?> completable) {  
         if (!this.isManualAck) {  
            this.logger.warn("Container AcknowledgeMode must be MANUAL for a Future<?> return type; "  
                  + "otherwise the container will ack the message immediately");  
         }  
         completable.whenComplete((r, t) -> {  
               if (t == null) {  
                  asyncSuccess(resultArg, request, channel, source, r);  
                  basicAck(request, channel);  
               }  
               else {  
                  asyncFailure(request, channel, t, source);  
               }  
         });  
      }  
      else if (monoPresent && MonoHandler.isMono(resultArg.getReturnValue())) {  
         if (!this.isManualAck) {  
            this.logger.warn("Container AcknowledgeMode must be MANUAL for a Mono<?> return type" +  
                  "(or Kotlin suspend function); otherwise the container will ack the message immediately");  
         }  
         MonoHandler.subscribe(resultArg.getReturnValue(),  
               r -> asyncSuccess(resultArg, request, channel, source, r),  
               t -> asyncFailure(request, channel, t, source),  
               () -> basicAck(request, channel));  
      }  
      else {  
         doHandleResult(resultArg, request, channel, source);  
      }  
   }  
   else if (this.logger.isWarnEnabled()) {  
      this.logger.warn("Listener method returned result [" + resultArg  
            + "]: not generating response message for it because no Rabbit Channel given");  
   }  
}
```