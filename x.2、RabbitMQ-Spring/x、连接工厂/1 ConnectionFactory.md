

# CachingConnectionFactory

默认情况，该类维护1个共享的Connection，缓存Channel（默认缓存25个，通过setChannelCacheSize()方法来设置）。

## 缓存Channel

默认情况下，该类只缓存Channel，不缓存Connection。

1）缓存大小：1.6之前为1，1.6之后为25。

2）理解缓存大小：默认情况下，缓存大小不是一个限制，而仅仅是Cache中可以缓存的Channel的最大数量。如缓存大小为10，实际上可以使用任意数量的Channel。如果使用了10个以上的通道，并且它们都返回到缓存中，那么将有10个Channel放入Cache中。其余部分会被关闭。

3）限制可用Channel数量：
- 如2中所述，默认情况 `channelCacheSize` 只是用来简单地限制Channel Cache的大小，对实际能够使用的Channel数量没有限制。
- 1.4.2以后，设置 `channelCheckoutTimeout`（setChannelCheckoutTimeout(long channelCheckoutTimeout)），`channelCacheSize` 语义会发生变化，能够限制每个Connection可用的Channel总数，当达到上限后，要求使用Channel的线程会被阻塞，直到有Channel可用或达到指定的时长channelCheckoutTimeout。
- 此时改变 `channelCacheSize` 大小，并不会影响已经存在的Connection，调用destroy()来创建一个新的Connection以让新的限制生效。

4）警告：在框架内部使用Channel（如RabbitTemplate）是可以保证Channel最终返回到Cache中的，但如果在框架外部使用（如直接通过Connection创建Channel），那么需要保证自己将其返回到Cache中（如在finnally块中），否则会耗尽可用的Channel。


## 缓存Connection

默认情况下，该类不缓存Connection，维护一个全应用共享的Connection，从1.3开始，能配置该类缓存Connection的功能。

1）配置缓存Connection：设置 `cacheMode` 为 `CacheMode.CONNECTION`
2）情境：
- 创建Connection（createConnection()）时会创建一个新的Connection或从Cache中取一个。
- 关闭Connection时，如果Connection Cache没有达到上限，则将Connection退回Cache中。
3）Connection Cache大小：
- 默认只限制允许多少idle Connection，不限制Connection的数量。
- 1.5.5以后，设置 `connectionLimit` 来限制Connection的数量，达到上限后，再创建Connection需要等到一个Connection变得idle，如果等待指定时长仍然没有，则抛出 `AmqpTimeoutException`。
4）重要：
- When the cache mode is CONNECTION, automatic declaration of queues and others (See Automatic Declaration of Exchanges, Queues, and Bindings) is NOT supported.
- Also, at the time of this writing, the amqp-client library by default creates a fixed thread pool for each connection (default size: Runtime.getRuntime().availableProcessors() * 2 threads). When using a large number of connections, you should consider setting a custom executor on the CachingConnectionFactory. Then, the same executor can be used by all connections and its threads can be shared. The executor’s thread pool should be unbounded or set appropriately for the expected use (usually, at least one thread per connection). If multiple channels are created on each connection, the pool size affects the concurrency, so a variable (or simple cached) thread pool executor would be most suitable.




```java
/* ------------------------------------- CachingConnectionFactory -------------------------------------
 */
public final Connection createConnection() throws AmqpException {  
	
    if (this.stopped) {  
        throw new AmqpApplicationContextClosedException(  
            "The ApplicationContext is closed and the ConnectionFactory can no longer create connections.");  
    }  
    // this.connectionMonitor:
    // - 默认(初)值为 new Object()
    synchronized (this.connectionMonitor) {  
        if (this.cacheMode == CacheMode.CHANNEL) {  
            if (this.connection.target == null) {  
	            this.connection.target = super.createBareConnection();  
	            // invoke the listener *after* this.connection is assigned  
	            if (!this.checkoutPermits.containsKey(this.connection)) {  
	                this.checkoutPermits.put(this.connection, new Semaphore(this.channelCacheSize));  
                }  
	            this.connection.closeNotified.set(false);  
	            getConnectionListener().onCreate(this.connection);  
            }  
            return this.connection;  
        }  
        else if (this.cacheMode == CacheMode.CONNECTION) {  
            return connectionFromCache();  
        }  
    }  
    return null; // NOSONAR - never reach here - exceptions  
}
```


# PooledChannelConnectionFactory

内含：
- 1个Connection
- 2个Channel Pool。一个用于事务Channel，一个用于非事务Channel。

针对Pool的说明：
- 该Pool是默认配置的GenericObjectPool。因为基于Apache Pool 2，需要导入 `commons-pool2` jar包。
- 提供一个Callback来配置Pooling。

```java
@Bean 
PooledChannelConnectionFactory pcf() throws Exception { 
	ConnectionFactory rabbitConnectionFactory = new ConnectionFactory(); 
	rabbitConnectionFactory.setHost("localhost"); 
	PooledChannelConnectionFactory pcf = new PooledChannelConnectionFactory(rabbitConnectionFactory); 
	pcf.setPoolConfigurer((pool, tx) -> { 
		if (tx) { 
			// configure the transactional pool 
		}
		else{ 
			// configure the non-transactional pool
		}
	});
	return pcf;
}
```

# ThreadChannelConnectionFactory


# SingleConnectionFactory

特点：只使用单元测试，不适用于生产环境。