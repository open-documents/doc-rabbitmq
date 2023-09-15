

```java
@Configuration(proxyBeanMethods = false)  
protected static class RabbitConnectionFactoryCreator {  
  
	private final RabbitProperties properties;  
  
	protected RabbitConnectionFactoryCreator(
		RabbitProperties properties,  
        ObjectProvider<RabbitConnectionDetails> connectionDetails) {  
        this.properties = properties;  
    }  

	// 该类由springboot提供
	@Bean  
    @ConditionalOnMissingBean(RabbitConnectionDetails.class)  
    RabbitConnectionDetails rabbitConnectionDetails() {  
		return new PropertiesRabbitConnectionDetails(this.properties);  
    }  

	/* 
	 *
	 *
	 */
    @Bean  
    @ConditionalOnMissingBean      
    RabbitConnectionFactoryBeanConfigurer rabbitConnectionFactoryBeanConfigurer(
	    ResourceLoader resourceLoader,  
        RabbitConnectionDetails connectionDetails, 
        ObjectProvider<CredentialsProvider> credentialsProvider,  
        ObjectProvider<CredentialsRefreshService> credentialsRefreshService) {  

        RabbitConnectionFactoryBeanConfigurer configurer = new RabbitConnectionFactoryBeanConfigurer(
	        resourceLoader,  
            this.properties, 
            connectionDetails
        );  
        configurer.setCredentialsProvider(credentialsProvider.getIfUnique());  
        configurer.setCredentialsRefreshService(credentialsRefreshService.getIfUnique());  
        return configurer;  
    }  


	
    @Bean  
    @ConditionalOnMissingBean      
    CachingConnectionFactoryConfigurer rabbitConnectionFactoryConfigurer(
	    RabbitConnectionDetails connectionDetails,  
        ObjectProvider<ConnectionNameStrategy> connectionNameStrategy) {  

        CachingConnectionFactoryConfigurer configurer = new CachingConnectionFactoryConfigurer(
	        this.properties,  
            connectionDetails);  
        configurer.setConnectionNameStrategy(connectionNameStrategy.getIfUnique());  
        return configurer;  
    }  

	/* 
	 *
	 */
    @Bean  
    @ConditionalOnMissingBean(ConnectionFactory.class)  
    CachingConnectionFactory rabbitConnectionFactory(  
        RabbitConnectionFactoryBeanConfigurer rabbitConnectionFactoryBeanConfigurer,  
        CachingConnectionFactoryConfigurer rabbitCachingConnectionFactoryConfigurer,  
        ObjectProvider<ConnectionFactoryCustomizer> connectionFactoryCustomizers) throws Exception {  

		// RabbitConnectionFactoryBean由spring提供
        RabbitConnectionFactoryBean connectionFactoryBean = new RabbitConnectionFactoryBean();  
        rabbitConnectionFactoryBeanConfigurer.configure(connectionFactoryBean);  
        connectionFactoryBean.afterPropertiesSet();  

        com.rabbitmq.client.ConnectionFactory connectionFactory = connectionFactoryBean.getObject();  
        connectionFactoryCustomizers.orderedStream()  
           .forEach((customizer) -> customizer.customize(connectionFactory));  
  
        CachingConnectionFactory factory = new CachingConnectionFactory(connectionFactory);  
        rabbitCachingConnectionFactoryConfigurer.configure(factory);  
  
        return factory;  
    }  
  
}  
```


```java

@Configuration(proxyBeanMethods = false)  
@Import(RabbitConnectionFactoryCreator.class)  
protected static class RabbitTemplateConfiguration {  
	// RabbitTemplateConfigurer由springboot提供
	@Bean  
    @ConditionalOnMissingBean
    public RabbitTemplateConfigurer rabbitTemplateConfigurer(
	    RabbitProperties properties,  
        ObjectProvider<MessageConverter> messageConverter,  
        ObjectProvider<RabbitRetryTemplateCustomizer> retryTemplateCustomizers) {  
	    RabbitTemplateConfigurer configurer = new RabbitTemplateConfigurer(properties);  
	    configurer.setMessageConverter(messageConverter.getIfUnique());  
        configurer.setRetryTemplateCustomizers(retryTemplateCustomizers.orderedStream().toList());  
        return configurer;  
    }  

	// RabbitTemplate由spring部分提供
    @Bean  
    @ConditionalOnSingleCandidate(ConnectionFactory.class)  
    @ConditionalOnMissingBean(RabbitOperations.class)  
    public RabbitTemplate rabbitTemplate(
	    RabbitTemplateConfigurer configurer, 
	    ConnectionFactory connectionFactory,  
        ObjectProvider<RabbitTemplateCustomizer> customizers) {  
        RabbitTemplate template = new RabbitTemplate();  
        configurer.configure(template, connectionFactory);  
        customizers.orderedStream().forEach((customizer) -> customizer.customize(template));  
        return template;  
    }  

	// AmqpAdmin由spring部分提供
    @Bean  
    @ConditionalOnSingleCandidate(ConnectionFactory.class)  
    @ConditionalOnProperty(prefix = "spring.rabbitmq", name = "dynamic", matchIfMissing = true)  
    @ConditionalOnMissingBean  
    public AmqpAdmin amqpAdmin(ConnectionFactory connectionFactory) {  
        return new RabbitAdmin(connectionFactory);  
    }  
  
}  
```

```java

@Configuration(proxyBeanMethods = false)  
@ConditionalOnClass(RabbitMessagingTemplate.class)  
@ConditionalOnMissingBean(RabbitMessagingTemplate.class)  
@Import(RabbitTemplateConfiguration.class)  
protected static class MessagingTemplateConfiguration {  
  
	@Bean  
    @ConditionalOnSingleCandidate(RabbitTemplate.class)  
    public RabbitMessagingTemplate rabbitMessagingTemplate(RabbitTemplate rabbitTemplate) {  
	    return new RabbitMessagingTemplate(rabbitTemplate);  
    }  

}  
```