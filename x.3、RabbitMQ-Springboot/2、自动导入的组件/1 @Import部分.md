
```java
@AutoConfiguration  
@ConditionalOnClass({ RabbitTemplate.class, Channel.class })  
@EnableConfigurationProperties(RabbitProperties.class)  
@Import({ RabbitAnnotationDrivenConfiguration.class, RabbitStreamConfiguration.class })  
public class RabbitAutoConfiguration {
	... //
}
```