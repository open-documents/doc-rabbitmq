
MessageListenerAdapter与前面提到的消息监听器组件相比有下面的优势：

1）业务逻辑和Message API耦合度更低。

2）能够向Broker发送Response消息。



# 一、核心方法（onMessage）

下面是该类的定义及核心方法。
```java
public class MessageListenerAdapter extends AbstractAdaptableMessageListener {

	public void onMessage(Message message, Channel channel) throws Exception { // NOSONAR  

		// 1. getDelegate(): 返回this.delegate
		// 2. this.delegate: 类型 Object
	    Object delegateListener = getDelegate();  
	    if (!delegateListener.equals(this)) {  
		    // 
	        if (delegateListener instanceof ChannelAwareMessageListener chaml) {  
	            chaml.onMessage(message, channel);  
	            return;    
			}  
	        else if (delegateListener instanceof MessageListener messageListener) {  
	            messageListener.onMessage(message);  
	            return;   
			}  
	    }  
  
	    // Regular case: find a handler method reflectively.  

		// 
	    Object convertedMessage = extractMessage(message);  

	    String methodName = getListenerMethodName(message, convertedMessage);  

	    if (methodName == null) {  
	        throw new AmqpIllegalStateException(...);  
	    }  
  
	    // Invoke the handler method with appropriate arguments.  
	    Object[] listenerArguments = buildListenerArguments(convertedMessage, channel, message); 

	    Object result = invokeListenerMethod(methodName, listenerArguments, message);  
	    if (result != null) {  
	        handleResult(new InvocationResult(result, null, null, null, null), message, channel);  
	    }  
	    else {  
	        ... // log
	    }  
	}
}
```
1 方法 -- extractMessage()
```java
/* -------------------------------------- AbstractAdaptableMessageListener --------------------------------------
 */
protected Object extractMessage(Message message) {  
   MessageConverter converter = getMessageConverter();  
   if (converter != null) {  
      return converter.fromMessage(message);  
   }  
   return message;  
}
```
方法 -- invokeListenerMethod()
```java
protected Object invokeListenerMethod(String methodName, Object[] arguments, Message originalMessage) {  
    try {  
	    // org.springframework.util包下的类
        MethodInvoker methodInvoker = new MethodInvoker();  
        // 1. 设置 methodInvoker#targetObject属性(Object类型)
        // 2. 如果targetObject不为null, 设置 methodInvoker#targetClass属性(Class属性,值为targetObject.getClass())
        methodInvoker.setTargetObject(getDelegate());  
	    // 设置 methodInvoker#targetMethod属性(String类型)
        methodInvoker.setTargetMethod(methodName);  
        // 设置 methodInvoker#arguments属性(Object[]类型)
        methodInvoker.setArguments(arguments);  
        
        methodInvoker.prepare();  
        return methodInvoker.invoke();  
    }  
    catch (InvocationTargetException ex) { ...  }  
    catch (Exception ex) { ... }
```

# 反射调用方法部分

## prepare()
```java
// ------------------------------------ MethodInvoker ------------------------------------
public void prepare() throws ClassNotFoundException, NoSuchMethodException {  
	// this.staticMethod:
	// 类型 String
	// 默认初值 null
    if (this.staticMethod != null) {  
        int lastDotIndex = this.staticMethod.lastIndexOf('.');  
        if (lastDotIndex == -1 || lastDotIndex == this.staticMethod.length()) {  
         throw new IllegalArgumentException(  
               "staticMethod must be a fully qualified class plus method name: " +  
               "e.g. 'example.MyExampleClass.myExampleMethod'");  
        }  
        String className = this.staticMethod.substring(0, lastDotIndex);  
        String methodName = this.staticMethod.substring(lastDotIndex + 1);  
        this.targetClass = resolveClassName(className);  
        this.targetMethod = methodName;  
    }  
  
    Class<?> targetClass = getTargetClass();  
    String targetMethod = getTargetMethod();  

	... // Assert
  
    Object[] arguments = getArguments();  
    Class<?>[] argTypes = new Class<?>[arguments.length];  
    for (int i = 0; i < arguments.length; ++i) {  
        argTypes[i] = (arguments[i] != null ? arguments[i].getClass() : Object.class);  
    }  
  
    // Try to get the exact method first.  
    try {  
        this.methodObject = targetClass.getMethod(targetMethod, argTypes);  
    }  
    catch (NoSuchMethodException ex) {  
        // Just rethrow exception if we can't get any match.  
        this.methodObject = findMatchingMethod();  
        if (this.methodObject == null) {  
            throw ex;  
        }  
    }  
}
```

```java
// ------------------------------------ MethodInvoker ------------------------------------
protected Method findMatchingMethod() {  
    String targetMethod = getTargetMethod();  
    Object[] arguments = getArguments();  
    int argCount = arguments.length;  
  
    Class<?> targetClass = getTargetClass();  
	... // Assert
	
    Method[] candidates = ReflectionUtils.getAllDeclaredMethods(targetClass);  
    int minTypeDiffWeight = Integer.MAX_VALUE;  
    Method matchingMethod = null;  
  
   for (Method candidate : candidates) {  
      if (candidate.getName().equals(targetMethod)) {  
         if (candidate.getParameterCount() == argCount) {  
            Class<?>[] paramTypes = candidate.getParameterTypes();  
            int typeDiffWeight = getTypeDifferenceWeight(paramTypes, arguments);  
            if (typeDiffWeight < minTypeDiffWeight) {  
               minTypeDiffWeight = typeDiffWeight;  
               matchingMethod = candidate;  
            }  
         }  
      }  
   }  
  
   return matchingMethod;  
}
```
## invoke()
```java
// ------------------------------------ MethodInvoker ------------------------------------
public Object invoke() throws InvocationTargetException, IllegalAccessException {  
    // In the static case, target will simply be {@code null}.  
    Object targetObject = getTargetObject();  
    Method preparedMethod = getPreparedMethod();  
    if (targetObject == null && !Modifier.isStatic(preparedMethod.getModifiers())) {  
        throw new IllegalArgumentException("Target method must not be non-static without a target");  
    }  
    ReflectionUtils.makeAccessible(preparedMethod);  
    return preparedMethod.invoke(targetObject, getArguments());  
}
```

# 指定监听器调用方法（2种方式）

```java
/* -------------------------------------- MessageListenerAdapter --------------------------------------
 */
protected String getListenerMethodName(Message originalMessage, Object extractedMessage) {  
   if (this.queueOrTagToMethodName.size() > 0) {  
      MessageProperties props = originalMessage.getMessageProperties();  
      String methodName = this.queueOrTagToMethodName.get(props.getConsumerQueue());  
      if (methodName == null) {  
         methodName = this.queueOrTagToMethodName.get(props.getConsumerTag());  
      }  
      if (methodName != null) {  
         return methodName;  
      }  
   }  
   return getDefaultListenerMethod();  
}
```

1）静态指定业务代码。
```java
MessageListenerAdapter listener = new MessageListenerAdapter(somePojo); 
listener.setDefaultListenerMethod("myMethod");
```

2）通过继承该类并重写`getListenerMethodName()`方法，动态指定业务代码。

# 为监听器方法传递参数

MessageListenerAdapter#buildListenerArguments()方法的返回值会作为参数传递到监听器方法中。

1）可以使用MessageListenerAdapter#buildListenerArguments()的默认逻辑。默认将转换后的Message传递给监听器方法。
```java
/* ----------------------------------- MessageListenerAdapter -----------------------------------
 */
protected Object[] buildListenerArguments(Object extractedMessage, Channel channel, Message message) {  
    return new Object[] { extractedMessage };  
}
```
2）也可以通过继承MessageListenerAdapter类，重写该方法来传递自定义参数给监听器方法。


# 向Broker发送Response




MessageListenerAdapter在使用自定义的监听器方法处理完消息后，还能够生成消息，并将消息发送到Broker。

主要涉及的处理方法：
- handleResult()



```java
@FunctionalInterface  
public interface ReplyingMessageListener<T, R> {  

   /**  
    * Handle the message and return a reply.    * @param t the request.  
    * @return the reply.  
    */   R handleMessage(T t);  
  
}

new MessageListenerAdapter((ReplyingMessageListener<String, String>) data -> {   ...   return result; }));
```


```java
/* ----------------------------------- AbstractAdaptableMessageListener -----------------------------------
 */
protected void handleResult(InvocationResult resultArg, Message request, Channel channel) {  
    handleResult(resultArg, request, channel, null);  
}
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

protected void doHandleResult(InvocationResult resultArg, Message request, Channel channel, Object source) {  
	... // log
    try {  
      Message response = buildMessage(channel, resultArg.getReturnValue(), resultArg.getReturnType());  
      MessageProperties props = response.getMessageProperties();  
      props.setTargetBean(resultArg.getBean());  
      props.setTargetMethod(resultArg.getMethod());  
      postProcessResponse(request, response);  
      if (this.replyPostProcessor != null) {  
         response = this.replyPostProcessor.apply(request, response);  
      }  
      Address replyTo = getReplyToAddress(request, source, resultArg);  
      sendResponse(channel, replyTo, response);  
   }  
   catch (Exception ex) {  
      throw new ReplyFailureException("Failed to send reply with payload '" + resultArg + "'", ex);  
   }  
}
```
## 获取发送消息的地址

getReplyToAddress()方法决定了发送返回消息到何处。
```java
protected Address getReplyToAddress(Message request, Object source, InvocationResult result) {  
    Address replyTo = request.getMessageProperties().getReplyToAddress();  
    if (replyTo == null) {  
        if (this.responseAddress == null && this.responseExchange != null) {  
            this.responseAddress = new Address(this.responseExchange, this.responseRoutingKey);  
        }  
        if (result.getSendTo() != null) {  
            replyTo = evaluateReplyTo(request, source, result.getReturnValue(), result.getSendTo());  
        }  
        else if (this.responseExpression != null) {  
            replyTo = evaluateReplyTo(request, source, result.getReturnValue(), this.responseExpression);  
        }  
        else if (this.responseAddress == null) {  
            throw new AmqpException(  
               "Cannot determine ReplyTo message property value: " +  
                     "Request message does not contain reply-to property, " +  
                     "and no default response Exchange was set.");  
        }  
        else {  
            replyTo = this.responseAddress;  
        }  
    }  
   return replyTo;  
}
```