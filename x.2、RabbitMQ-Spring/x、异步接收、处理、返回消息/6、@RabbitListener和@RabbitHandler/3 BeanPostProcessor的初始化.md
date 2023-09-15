
```java 
/* ------------------------------------ RabbitListenerAnnotationBeanPostProcessor ------------------------------------
 */
public void afterSingletonsInstantiated() {  

	// this.registrar:
	// - 类型为 RabbitListenerEndpointRegistrar
	// - 默认(初)值为 new RabbitListenerEndpointRegistrar()
    this.registrar.setBeanFactory(this.beanFactory);  
  
    if (this.beanFactory instanceof ListableBeanFactory lbf) {  
        Map<String, RabbitListenerConfigurer> instances = lbf.getBeansOfType(RabbitListenerConfigurer.class);  
        for (RabbitListenerConfigurer configurer : instances.values()) {  
            configurer.configureRabbitListeners(this.registrar);  
        }  
    }  
  
    if (this.registrar.getEndpointRegistry() == null) {  
        if (this.endpointRegistry == null) {  
	        ... // assert
            this.endpointRegistry = this.beanFactory.getBean( 
	            // "org.springframework.amqp.rabbit.config.internalRabbitListenerEndpointRegistry"
                RabbitListenerConfigUtils.RABBIT_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME,  
                RabbitListenerEndpointRegistry.class
            );
        }  
        this.registrar.setEndpointRegistry(this.endpointRegistry);  
    }  


	// 
	// this.defaultContainerFactoryBeanName:
	// - 默认(初)值为 "rabbitListenerContainerFactory"
    if (this.defaultContainerFactoryBeanName != null) {  
        this.registrar.setContainerFactoryBeanName(this.defaultContainerFactoryBeanName);  
    }  
  
    // Set the custom handler method factory once resolved by the configurer  
    MessageHandlerMethodFactory handlerMethodFactory = this.registrar.getMessageHandlerMethodFactory();  
    if (handlerMethodFactory != null) {  
        this.messageHandlerMethodFactory.setMessageHandlerMethodFactory(handlerMethodFactory);  
    }  
  
    // Actually register all listeners  
    this.registrar.afterPropertiesSet();
  
    // clear the cache - prototype beans will be re-cached.  
    this.typeCache.clear();  
}
```


```java
/* ------------------------------------ RabbitListenerEndpointRegistrar ------------------------------------
 */
public void afterPropertiesSet() {  
    registerAllEndpoints();  
}

protected void registerAllEndpoints() {  
	... // Assert

    synchronized (this.endpointDescriptors) {  
      for (AmqpListenerEndpointDescriptor descriptor : this.endpointDescriptors) {  
         if (descriptor.endpoint instanceof MultiMethodRabbitListenerEndpoint multi && this.validator != null) {  
            multi.setValidator(this.validator);  
         }  
         this.endpointRegistry.registerListenerContainer(// NOSONAR never null  
               descriptor.endpoint, resolveContainerFactory(descriptor));  
      }  
      this.startImmediately = true;  // trigger immediate startup  
   }  
}
```


```java
public void registerEndpoint(RabbitListenerEndpoint endpoint, @Nullable RabbitListenerContainerFactory<?> factory) {  
	... // Assert
	
    // Factory may be null, we defer the resolution right before actually creating the container  
    // AmqpListenerEndpointDescriptor是对RabbitListenerEndpoint和RabbitListenerContainerFactory的简单包装
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
