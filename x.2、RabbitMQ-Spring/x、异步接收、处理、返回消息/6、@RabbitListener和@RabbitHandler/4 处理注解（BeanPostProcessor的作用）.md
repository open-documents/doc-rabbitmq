

# 一、（重要）对@RabbitListener和@RabbitHandler的处理（Bean初始化后的处理）

```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 * 描述: 该方法在RabbitListenerAnnotationBeanPostProcessor实例初始化完成后进行,即此方法被调用前,一定完成了afterSingletonsInstantiated()
 */
public Object postProcessAfterInitialization(final Object bean, final String beanName) throws BeansException {  

    Class<?> targetClass = AopUtils.getTargetClass(bean);  

	// 1. 
    // this.typeCache:
    // - 类型 ConcurrentMap<Class<?>, RabbitListenerAnnotationBeanPostProcessor.TypeMetadata>
    // - 默认(初)值 new ConcurrentHashMap<Class<?>, RabbitListenerAnnotationBeanPostProcessor.TypeMetadata>() 
    // 2. 
    final TypeMetadata metadata = this.typeCache.computeIfAbsent(targetClass, this::buildMetadata);      // 1

	// 遍历每个@RabbitListener方法
    for (ListenerMethod lm : metadata.listenerMethods) {  
	    // 遍历标注在该方法上的每个@RabbitListener注解
        for (RabbitListener rabbitListener : lm.annotations) {  
            processAmqpListener(rabbitListener, lm.method, bean, beanName);                              // 2
        }  
    }  
    if (metadata.handlerMethods.length > 0) {  
        processMultiMethodListeners(metadata.classAnnotations, metadata.handlerMethods, bean, beanName);  // 3
    }  
    return bean;  
}
```
## 1 获取@RabbitListener和@RabbitHandler元数据 -- buildMetadata()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private TypeMetadata buildMetadata(Class<?> targetClass) {  

	// 获取标注在类上的@RabbitListener注解
    List<RabbitListener> classLevelListeners = findListenerAnnotations(targetClass); 
    final boolean hasClassLevelListeners = classLevelListeners.size() > 0;  

    final List<ListenerMethod> methods = new ArrayList<>();  
    final List<Method> multiMethods = new ArrayList<>();  

	// 对每个满足条件的方法进行相同的处理
    ReflectionUtils.doWithMethods(
	    targetClass, 
	    method -> {  
		    // 获取标注在方法上的@RabbitListener注解
	        List<RabbitListener> listenerAnnotations = findListenerAnnotations(method);  
	        if (listenerAnnotations.size() > 0) {  
	            methods.add(new ListenerMethod(
		            method,
	                listenerAnnotations.toArray(new RabbitListener[listenerAnnotations.size()])
				));  
	        }  
	        // 类标注了@RabbitListener注解
	        if (hasClassLevelListeners) {  
	            RabbitHandler rabbitHandler = AnnotationUtils.findAnnotation(method, RabbitHandler.class);  
	            if (rabbitHandler != null) {  
                    multiMethods.add(method);  
	            }  
	        }  
        }, 
        // 过滤条件(同时满足):
        // 1. 
        // 2. 
        ReflectionUtils.USER_DECLARED_METHODS.and(meth -> !meth.getDeclaringClass().getName().contains("$MockitoMock$"))); 

    if (methods.isEmpty() && multiMethods.isEmpty()) {  
        return TypeMetadata.EMPTY;  
    }  
    return new TypeMetadata(  
         methods.toArray(new ListenerMethod[methods.size()]),  
         multiMethods.toArray(new Method[multiMethods.size()]),  
         classLevelListeners.toArray(new RabbitListener[classLevelListeners.size()]));  
}
```
## 2 processAmqpListener()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 * 参数:
 *   - rabbitListener: @RabbitListener注解
 *   - method: @RabbitListener方法
 *   - bean: 被BeanPostProcessor处理的Bean
 *   - beanName: Bean名称
 */
protected Collection<Declarable> processAmqpListener(RabbitListener rabbitListener, Method method, Object bean, String beanName) {  
    Method methodToUse = checkProxy(method, bean);  
    MethodRabbitListenerEndpoint endpoint = new MethodRabbitListenerEndpoint();  
    endpoint.setMethod(methodToUse);  
    return processListener(endpoint, rabbitListener, bean, methodToUse, beanName);  
}
```
## 3 processMultiMethodListeners()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private void processMultiMethodListeners(RabbitListener[] classLevelListeners, Method[] multiMethods, Object bean, String beanName) {  
  
    List<Method> checkedMethods = new ArrayList<Method>();  
    Method defaultMethod = null;  
    for (Method method : multiMethods) {  
        Method checked = checkProxy(method, bean);  
        if (AnnotationUtils.findAnnotation(method, RabbitHandler.class).isDefault()) { // NOSONAR never null  
            final Method toAssert = defaultMethod;  
            // assert: "只能声明一个默认@RabbitHandler方法"
            defaultMethod = checked;  
        }  
        checkedMethods.add(checked);  
    }  

    for (RabbitListener classLevelListener : classLevelListeners) {  
        MultiMethodRabbitListenerEndpoint endpoint = new MultiMethodRabbitListenerEndpoint(checkedMethods, defaultMethod, bean);  
        processListener(endpoint, classLevelListener, bean, bean.getClass(), beanName);  
    }  
}
```

# 二、重要方法 -- processListener()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 * 参数:
 *   - endpoint:
 *   - rabbitListener:
 *   - bean: 被BeanPostProcessor处理的Bean
 *   - target: 可能是@RabbitListener方法,可能是bean实例的类
 *   - beanName: Bean名称
 */
protected Collection<Declarable> processListener(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener, Object bean, Object target, String beanName) {  
  
    final List<Declarable> declarables = new ArrayList<>();  

    endpoint.setBean(bean);  

    // this.messageHandlerMethodFactory:
    // - 类型 RabbitHandlerMethodFactoryAdapter
    // - 默认(初)值 new RabbitListenerAnnotationBeanPostProcessor.RabbitHandlerMethodFactoryAdapter()
    endpoint.setMessageHandlerMethodFactory(this.messageHandlerMethodFactory);  

	
    endpoint.setId(getEndpointId(rabbitListener));  

    List<Object> resolvedQueues = resolveQueues(rabbitListener, declarables);  // 1
    if (!resolvedQueues.isEmpty()) {  
        if (resolvedQueues.get(0) instanceof String) {  
            endpoint.setQueueNames(resolvedQueues.stream().map(o -> (String) o).toArray(String[]::new));  
        }   
        else {  
            endpoint.setQueues(resolvedQueues.stream().map(o -> (Queue) o).toArray(Queue[]::new));  
        }  
    }  

    endpoint.setConcurrency(resolveExpressionAsStringOrInteger(rabbitListener.concurrency(), "concurrency"));  
    endpoint.setBeanFactory(this.beanFactory);  
    endpoint.setReturnExceptions(resolveExpressionAsBoolean(rabbitListener.returnExceptions()));  
    resolveErrorHandler(endpoint, rabbitListener);  // 2


    String group = rabbitListener.group();  
    if (StringUtils.hasText(group)) {  
        Object resolvedGroup = resolveExpression(group);  
        if (resolvedGroup instanceof String str) {  
            endpoint.setGroup(str);  
        }  
    }  

    String autoStartup = rabbitListener.autoStartup();  
    if (StringUtils.hasText(autoStartup)) {  
        endpoint.setAutoStartup(resolveExpressionAsBoolean(autoStartup));  
    }  
  
    endpoint.setExclusive(rabbitListener.exclusive());  


    String priority = resolveExpressionAsString(rabbitListener.priority(), "priority");  
    if (StringUtils.hasText(priority)) {  
        try {  
            endpoint.setPriority(Integer.valueOf(priority));  
        }  
	    catch (NumberFormatException ex) {  
            throw new BeanInitializationException("Invalid priority value for (rabbitListener) must be an integer", ...);  
        }  
    }  
  
    resolveExecutor(endpoint, rabbitListener, target, beanName);           // 3
    resolveAdmin(endpoint, rabbitListener, target);                        // 4
    resolveAckMode(endpoint, rabbitListener);                              // 5
    resolvePostProcessor(endpoint, rabbitListener, target, beanName);      // 6
    resolveMessageConverter(endpoint, rabbitListener, target, beanName);   // 7
    resolveReplyContentType(endpoint, rabbitListener);                     // 8

    if (StringUtils.hasText(rabbitListener.batch())) {  
        endpoint.setBatchListener(Boolean.parseBoolean(rabbitListener.batch()));  
    }  

    RabbitListenerContainerFactory<?> factory = resolveContainerFactory(rabbitListener, target, beanName);   // 9
    this.registrar.registerEndpoint(endpoint, factory);   // 10

    return declarables;  
}
```
## 1 resolveQueues()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private List<Object> resolveQueues(RabbitListener rabbitListener, Collection<Declarable> declarables) {  

	String[] queues = rabbitListener.queues();  
    List<String> queueNames = new ArrayList<>();  
    List<Queue> queueBeans = new ArrayList<>();  
    for (String queue : queues) {  
        resolveQueues(queue, queueNames, queueBeans);  
    }  
    if (!queueNames.isEmpty()) {  
        // revert to the previous behavior of just using the name when there is mixture of String and Queue  
        queueBeans.forEach(qb -> queueNames.add(qb.getName()));  
        queueBeans.clear();  
    }  

    org.springframework.amqp.rabbit.annotation.Queue[] queuesToDeclare = rabbitListener.queuesToDeclare();  
    if (queuesToDeclare.length > 0) {  
        if (queues.length > 0) {  
            throw new BeanInitializationException("@RabbitListener只能有queues, queuesToDeclare, bindings属性中的一个");  
        }  
        for (org.springframework.amqp.rabbit.annotation.Queue queue : queuesToDeclare) {  
            queueNames.add(declareQueue(queue, declarables));  
        }  
    }  

    QueueBinding[] bindings = rabbitListener.bindings();  
    if (bindings.length > 0) {  
        if (queues.length > 0 || queuesToDeclare.length > 0) {  
            throw new BeanInitializationException("@RabbitListener只能有queues, queuesToDeclare, bindings属性中的一个"); 
        }  
        return Arrays.stream(registerBeansForDeclaration(rabbitListener, declarables))  
            .map(s -> (Object) s)  
            .collect(Collectors.toList());  
   }  
   return queueNames.isEmpty()  
         ? queueBeans.stream()  
               .map(s -> (Object) s)  
               .collect(Collectors.toList())  
         : queueNames.stream()  
               .map(s -> (Object) s)  
               .collect(Collectors.toList());  
  
}
```
### 方法 -- resolveQueues()
```java
/*
 * @RabbitListener(queues = {"a", })
 */
private void resolveQueues(String queue, List<String> result, List<Queue> queueBeans) {  
    resolveAsStringOrQueue(resolveExpression(queue), result, queueBeans, "queues");  
}

private void resolveAsStringOrQueue(Object resolvedValue, List<String> names, @Nullable List<Queue> queues, String what) {  
  
    Object resolvedValueToUse = resolvedValue;  
    if (resolvedValue instanceof String[] strings) {  
        resolvedValueToUse = Arrays.asList(strings);  
    }  
    if (queues != null && resolvedValueToUse instanceof Queue q) {  
        if (!names.isEmpty()) {  
            // revert to the previous behavior of just using the name when there is mixture of String and Queue  
            names.add(q.getName());  
        }  
        else {  
            queues.add(q);  
        }  
    }  
    else if (resolvedValueToUse instanceof String str) {  
        names.add(str);  
    }  
    else if (resolvedValueToUse instanceof Iterable) {  
        for (Object object : (Iterable<Object>) resolvedValueToUse) {  
            resolveAsStringOrQueue(object, names, queues, what);  
        }  
    }  
    else {  
        throw new IllegalArgumentException(...);  
    }  
}
```
### 方法 -- declareQueue()
```java
private String declareQueue(org.springframework.amqp.rabbit.annotation.Queue bindingQueue, Collection<Declarable> declarables) {

    String queueName = (String) resolveExpression(bindingQueue.value());  
    boolean isAnonymous = false;  
    if (!StringUtils.hasText(queueName)) {  
        queueName = Base64UrlNamingStrategy.DEFAULT.generateName();  
        // default exclusive/autodelete and non-durable when anonymous  
        isAnonymous = true;  
    }  

    Queue queue = new Queue(
	    queueName,  
        resolveExpressionAsBoolean(bindingQueue.durable(), !isAnonymous),  
        resolveExpressionAsBoolean(bindingQueue.exclusive(), isAnonymous),  
        resolveExpressionAsBoolean(bindingQueue.autoDelete(), isAnonymous),  
        resolveArguments(bindingQueue.arguments())
    );  
    queue.setIgnoreDeclarationExceptions(resolveExpressionAsBoolean(bindingQueue.ignoreDeclarationExceptions()));  

    ((ConfigurableBeanFactory) this.beanFactory).registerSingleton(queueName + ++this.increment, queue);  
    if (bindingQueue.admins().length > 0) {  
        queue.setAdminsThatShouldDeclare((Object[]) bindingQueue.admins());  
    }  

    queue.setShouldDeclare(resolveExpressionAsBoolean(bindingQueue.declare()));  
    declarables.add(queue);  
    return queueName;  
}
```
### 方法 -- registerBeansForDeclaration()
```java
private String[] registerBeansForDeclaration(RabbitListener rabbitListener, Collection<Declarable> declarables) {  
    List<String> queues = new ArrayList<String>();  
    if (this.beanFactory instanceof ConfigurableBeanFactory) {  
        for (QueueBinding binding : rabbitListener.bindings()) {  
            String queueName = declareQueue(binding.value(), declarables);  
            queues.add(queueName);  
            declareExchangeAndBinding(binding, queueName, declarables);  
        }  
    }  
    return queues.toArray(new String[0]);  
}

private void declareExchangeAndBinding(QueueBinding binding, String queueName, Collection<Declarable> declarables) {  
   org.springframework.amqp.rabbit.annotation.Exchange bindingExchange = binding.exchange();  
   String exchangeName = resolveExpressionAsString(bindingExchange.value(), "@Exchange.exchange");  
   Assert.isTrue(StringUtils.hasText(exchangeName), () -> "Exchange name required; binding queue " + queueName);  
   String exchangeType = resolveExpressionAsString(bindingExchange.type(), "@Exchange.type");  
  
   ExchangeBuilder exchangeBuilder = new ExchangeBuilder(exchangeName, exchangeType);  
  
   if (resolveExpressionAsBoolean(bindingExchange.autoDelete())) {  
      exchangeBuilder.autoDelete();  
   }  
  
   if (resolveExpressionAsBoolean(bindingExchange.internal())) {  
      exchangeBuilder.internal();  
   }  
  
   if (resolveExpressionAsBoolean(bindingExchange.delayed())) {  
      exchangeBuilder.delayed();  
   }  
  
   if (resolveExpressionAsBoolean(bindingExchange.ignoreDeclarationExceptions())) {  
      exchangeBuilder.ignoreDeclarationExceptions();  
   }  
  
   if (!resolveExpressionAsBoolean(bindingExchange.declare())) {  
      exchangeBuilder.suppressDeclaration();  
   }  
  
   if (bindingExchange.admins().length > 0) {  
      exchangeBuilder.admins((Object[]) bindingExchange.admins());  
   }  
  
   Map<String, Object> arguments = resolveArguments(bindingExchange.arguments());  
  
   if (!CollectionUtils.isEmpty(arguments)) {  
      exchangeBuilder.withArguments(arguments);  
   }  
  
  
   org.springframework.amqp.core.Exchange exchange =  
         exchangeBuilder.durable(resolveExpressionAsBoolean(bindingExchange.durable()))  
               .build();  
  
   ((ConfigurableBeanFactory) this.beanFactory)  
         .registerSingleton(exchangeName + ++this.increment, exchange);  
   registerBindings(binding, queueName, exchangeName, exchangeType, declarables);  
   declarables.add(exchange);  
}
```
## 2 resolveErrorHandler()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private void resolveErrorHandler(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener) {  
    Object errorHandler = resolveExpression(rabbitListener.errorHandler());  
    if (errorHandler instanceof RabbitListenerErrorHandler rleh) {  
        endpoint.setErrorHandler(rleh);  
    }  
    else {  
        String errorHandlerBeanName = resolveExpressionAsString(rabbitListener.errorHandler(), "errorHandler");  
        if (StringUtils.hasText(errorHandlerBeanName)) {  
            endpoint.setErrorHandler(this.beanFactory.getBean(errorHandlerBeanName, RabbitListenerErrorHandler.class));  
        }  
    }  
}
```

## 3 resolveExecutor()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private void resolveExecutor(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener, Object execTarget, String beanName) {  
    Object resolved = resolveExpression(rabbitListener.executor());  
    if (resolved instanceof TaskExecutor tex) {  
        endpoint.setTaskExecutor(tex);  
    }  
    else {  
        String execBeanName = resolveExpressionAsString(rabbitListener.executor(), "executor");  
        if (StringUtils.hasText(execBeanName)) {  
            ... // Assert
            try {  
	            endpoint.setTaskExecutor(this.beanFactory.getBean(execBeanName, TaskExecutor.class));  
            }  
            catch (NoSuchBeanDefinitionException ex) {  ...  }  
        }  
    }  
}
```

## 4 resolveAdmin()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private void resolveAdmin(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener, Object adminTarget) {  

    Object resolved = resolveExpression(rabbitListener.admin());  
    if (resolved instanceof AmqpAdmin admin) {  
        endpoint.setAdmin(admin);  
    }  

    else {  
        String rabbitAdmin = resolveExpressionAsString(rabbitListener.admin(), "admin");  
        if (StringUtils.hasText(rabbitAdmin)) {  
            try {  
	            endpoint.setAdmin(this.beanFactory.getBean(rabbitAdmin, RabbitAdmin.class));  
            }   
            catch (NoSuchBeanDefinitionException ex) {  ...  }  
        }  
    }  
}
```

## 5 resolveAckMode()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private void resolveAckMode(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener) {  
    String ackModeAttr = rabbitListener.ackMode();  
    if (StringUtils.hasText(ackModeAttr)) {  
        Object ackMode = resolveExpression(ackModeAttr);  
        if (ackMode instanceof String str) {  
            endpoint.setAckMode(AcknowledgeMode.valueOf(str));  
        }  
        else if (ackMode instanceof AcknowledgeMode mode) {  
            endpoint.setAckMode(mode);  
        }  
        else {  
            Assert.isNull(ackMode, "ackMode must resolve to a String or AcknowledgeMode");  
        }  
    }  
}
```

## 6 resolvePostProcessor()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private void resolvePostProcessor(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener, Object target, String beanName) {  
  
    Object resolved = resolveExpression(rabbitListener.replyPostProcessor());  
    if (resolved instanceof ReplyPostProcessor rpp) {  
        endpoint.setReplyPostProcessor(rpp);  
    }  
    else {  
        String ppBeanName = resolveExpressionAsString(rabbitListener.replyPostProcessor(), "replyPostProcessor");  
        if (StringUtils.hasText(ppBeanName)) {  
		    ... // Assert    
            try {  
	            endpoint.setReplyPostProcessor(this.beanFactory.getBean(ppBeanName, ReplyPostProcessor.class));  
            }  
            catch (NoSuchBeanDefinitionException ex) { ... }  
        }  
    }  
}
```

## 7 resolveMessageConverter()

```java
private void resolveMessageConverter(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener, Object target, String beanName) {  
  
    Object resolved = resolveExpression(rabbitListener.messageConverter());  
    if (resolved instanceof MessageConverter converter) {  
        endpoint.setMessageConverter(converter);  
    }  
    else {  
        String mcBeanName = resolveExpressionAsString(rabbitListener.messageConverter(), "messageConverter");  
        if (StringUtils.hasText(mcBeanName)) {  
            ... // assert: this.beanFactory!=null
            try {  
	            endpoint.setMessageConverter(this.beanFactory.getBean(mcBeanName, MessageConverter.class));  
            }  
            catch (NoSuchBeanDefinitionException ex) {  throw new BeanInitializationException(...);  }  
        }  
    }  
}
```


## 8 resolveReplyContentType()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private void resolveReplyContentType(MethodRabbitListenerEndpoint endpoint, RabbitListener rabbitListener) {  
    String contentType = resolveExpressionAsString(rabbitListener.replyContentType(), "replyContentType");  
    if (StringUtils.hasText(contentType)) {  
        endpoint.setReplyContentType(contentType);  
        endpoint.setConverterWinsContentType(resolveExpressionAsBoolean(rabbitListener.converterWinsContentType()));  
    }  
}
```
## 9 resolveContainerFactory()
```java
/* -------------------------------- RabbitListenerAnnotationBeanPostProcessor --------------------------------
 */
private RabbitListenerContainerFactory<?> resolveContainerFactory(RabbitListener rabbitListener, Object factoryTarget, String beanName) {  
  
    RabbitListenerContainerFactory<?> factory = null; 

	// 尝试将@RabbitListener的containerFactory()属性值解析为RabbitListenerContainerFactory对象
	Object resolved = resolveExpression(rabbitListener.containerFactory());  
    if (resolved instanceof RabbitListenerContainerFactory<?> rlcf) {  
        return rlcf;  
    }  

	// 尝试将@RabbitListener的containerFactory()属性值解析为String类型值
    String containerFactoryBeanName = resolveExpressionAsString(rabbitListener.containerFactory(), "containerFactory");  
    if (StringUtils.hasText(containerFactoryBeanName)) {  
        ... // Assert
        try {  
	        // 尝试从BeanFactory中获取
            factory = this.beanFactory.getBean(containerFactoryBeanName, RabbitListenerContainerFactory.class);  
        }  
        catch (NoSuchBeanDefinitionException ex) {  ...  }  
    return factory;  
}
```


# 10 registerEndpoint()
```java
public void registerEndpoint(RabbitListenerEndpoint endpoint, @Nullable RabbitListenerContainerFactory<?> factory) {  

	... // assert

    // Factory may be null, we defer the resolution right before actually creating the container  
    AmqpListenerEndpointDescriptor descriptor = new AmqpListenerEndpointDescriptor(endpoint, factory);  
    synchronized (this.endpointDescriptors) {  
        if (this.startImmediately) { // Register and start immediately  
            this.endpointRegistry.registerListenerContainer(
	            descriptor.endpoint, // NOSONAR never null  
                resolveContainerFactory(descriptor), true);  
        }  
        else {  
            this.endpointDescriptors.add(descriptor);  
        }  
    }  
}
```

## registerListenerContainer()
```java
/* ---------------------------------- RabbitListenerEndpointRegistrar ----------------------------------
 */
public void registerListenerContainer(RabbitListenerEndpoint endpoint, RabbitListenerContainerFactory<?> factory,  
         boolean startImmediately) {  

	... // assert
  
    String id = endpoint.getId();  
	... // assert

	// this.listenerContainers:
	// 类型 Map<String, MessageListenerContainer> listenerContainers
	// 默认初值 new ConcurrentHashMap<String, MessageListenerContainer>()
    synchronized (this.listenerContainers) {  
		... // assert 
        MessageListenerContainer container = createListenerContainer(endpoint, factory);  
        this.listenerContainers.put(id, container);  

        if (StringUtils.hasText(endpoint.getGroup()) && this.applicationContext != null) {  
            List<MessageListenerContainer> containerGroup;  
            if (this.applicationContext.containsBean(endpoint.getGroup())) {  
	            containerGroup = this.applicationContext.getBean(endpoint.getGroup(), List.class);  
            }  
            else {  
	            containerGroup = new ArrayList<MessageListenerContainer>();  
	            this.applicationContext.getBeanFactory().registerSingleton(endpoint.getGroup(), containerGroup);  
            }  
            containerGroup.add(container);  
        }  
        if (this.contextRefreshed) {  
            container.lazyLoad();  
        }  
        if (startImmediately) {  
            startIfNecessary(container);  
        }  
    }  
}
```
## 创建监听器容器 -- createListenerContainer()
```java
protected MessageListenerContainer createListenerContainer(RabbitListenerEndpoint endpoint, RabbitListenerContainerFactory<?> factory) {  

	// 因为监听器容器的创建很复杂,所以单独在第4节展开介绍
    MessageListenerContainer listenerContainer = factory.createListenerContainer(endpoint);  
    listenerContainer.afterPropertiesSet();  
  
    int containerPhase = listenerContainer.getPhase();  
    if (containerPhase < Integer.MAX_VALUE) {  // a custom phase value  
      if (this.phase < Integer.MAX_VALUE && this.phase != containerPhase) {  
         throw new IllegalStateException("Encountered phase mismatch between container factory definitions: " +  
               this.phase + " vs " + containerPhase);  
      }  
      this.phase = listenerContainer.getPhase();  
   }  
  
   return listenerContainer;  
}
```



# 帮助方法

1）
```java
protected String resolveExpressionAsString(String value, String attribute) {  
    Object resolved = resolveExpression(value);  
    if (resolved instanceof String str) {  
        return str;  
    }  
    else {  
        throw new IllegalStateException("The [" + attribute + "] must resolve to a String. "  
            + "Resolved to [" + resolved.getClass() + "] for [" + value + "]");  
    }  
}
```
2）
```java
private String resolveExpressionAsStringOrInteger(String value, String attribute) {  
   if (!StringUtils.hasLength(value)) {  
      return null;  
   }  
   Object resolved = resolveExpression(value);  
   if (resolved instanceof String str) {  
      return str;  
   }  
   else if (resolved instanceof Integer) {  
      return resolved.toString();  
   }  
   else {  
      throw new IllegalStateException("The [" + attribute + "] must resolve to a String. "  
            + "Resolved to [" + resolved.getClass() + "] for [" + value + "]");  
   }  
}
```
3）
```java
private Object resolveExpression(String value) {  
    String resolvedValue = resolve(value);  
   
    return this.resolver.evaluate(resolvedValue, this.expressionContext);  
}

private String resolve(String value) {  
    if (this.beanFactory != null && this.beanFactory instanceof ConfigurableBeanFactory cbf) {  
        return cbf.resolveEmbeddedValue(value);  
    }  
    return value;  
}
```

