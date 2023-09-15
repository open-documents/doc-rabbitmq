
# 一、BeanPostProcessor的导入

```java
@Order  
public class RabbitListenerConfigurationSelector implements DeferredImportSelector {  
  
    @Override  
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {  
        return new String[] {
            MultiRabbitBootstrapConfiguration.class.getName(),  
            RabbitBootstrapConfiguration.class.getName()
        };  
    }  
  
}
```

```java
public class RabbitBootstrapConfiguration implements ImportBeanDefinitionRegistrar {  
  
    @Override  
    public void registerBeanDefinitions(@Nullable AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) { 

		// 如果容器中不存在 名称为internalRabbitListenerAnnotationProcessor的BeanDefinition
		// 则向容器中注册 RabbitListenerAnnotationBeanPostProcessor.class对应的BeanDefinition
		// - 键: RabbitListenerConfigUtils.RABBIT_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME
		// - BeanDefinition名称: "org.springframework.amqp.rabbit.config.internalRabbitListenerAnnotationProcessor"
	    if (!registry.containsBeanDefinition(RabbitListenerConfigUtils.RABBIT_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME)) {  
		    registry.registerBeanDefinition(
			    RabbitListenerConfigUtils.RABBIT_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME,  
                new RootBeanDefinition(RabbitListenerAnnotationBeanPostProcessor.class)
            );  
        }  

		// 如果容器中不存在 名称为internalRabbitListenerEndpointRegistry的BeanDefinition
		// 则向容器中注册 RabbitListenerEndpointRegistry.class对应的BeanDefinition
		// - 常量: RabbitListenerConfigUtils.RABBIT_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME
		// - BeanDefinition名称: "org.springframework.amqp.rabbit.config.internalRabbitListenerEndpointRegistry"
        if (!registry.containsBeanDefinition(RabbitListenerConfigUtils.RABBIT_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME)) {  
            registry.registerBeanDefinition(
	            RabbitListenerConfigUtils.RABBIT_LISTENER_ENDPOINT_REGISTRY_BEAN_NAME,  
                new RootBeanDefinition(RabbitListenerEndpointRegistry.class)
            );  
        }  
    }  
}
```

```java
public class MultiRabbitBootstrapConfiguration implements ImportBeanDefinitionRegistrar, EnvironmentAware {

	...

	public void registerBeanDefinitions(@Nullable AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {  
		// 1. isMultiRabbitEnabled(): Environment实例中包含 "spring.multirabbitmq.enabled" 属性
		// 2. 容器中不包含 "org.springframework.amqp.rabbit.config.internalRabbitListenerAnnotationProcessor"对应的BeanDefinition 
	    if (isMultiRabbitEnabled() 
	        && !registry.containsBeanDefinition(RabbitListenerConfigUtils.RABBIT_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME)) {  
	        // 向容器中注册 MultiRabbitListenerAnnotationBeanPostProcessor.class对应的BeanDefinition
		    registry.registerBeanDefinition(
			    RabbitListenerConfigUtils.RABBIT_LISTENER_ANNOTATION_PROCESSOR_BEAN_NAME,  
	            new RootBeanDefinition(MultiRabbitListenerAnnotationBeanPostProcessor.class)
	        );  
	    }  
	}

	...
}
```