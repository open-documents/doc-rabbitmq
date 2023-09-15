








ConnectionFactory提供创建Connection的诸多newConnection()重载方法，下面作一个汇总。

<font color=44cf57>所有的newConnection()重载方法都抛出 IOException 和 TimeoutException</font> 。

<font color="00E0FF">1）第一部分方法</font>

这些方法都使用第二部分方法1。
```java
public Connection newConnection(Address[] addrs) .. {  
    return newConnection(this.sharedExecutor, Arrays.asList(addrs), null);  
}
public Connection newConnection(Address[] addrs, String clientProvidedName) .. {  
    return newConnection(this.sharedExecutor, Arrays.asList(addrs), clientProvidedName);  
}
public Connection newConnection(List<Address> addrs) .. {  
    return newConnection(this.sharedExecutor, addrs, null);  
}
public Connection newConnection(List<Address> addrs, String clientProvidedName) .. {  
    return newConnection(this.sharedExecutor, addrs, clientProvidedName);  
}
public Connection newConnection(ExecutorService executor, Address[] addrs) .. {  
    return newConnection(executor, Arrays.asList(addrs), null);  
}
public Connection newConnection(ExecutorService executor, Address[] addrs) .. {  
    return newConnection(executor, Arrays.asList(addrs), null);  
}
public Connection newConnection(ExecutorService executor, Address[] addrs, String clientProvidedName) .. {  
    return newConnection(executor, Arrays.asList(addrs), clientProvidedName);  
}
public Connection newConnection(ExecutorService executor, List<Address> addrs) .. {  
    return newConnection(executor, addrs, null);  
}
public Connection newConnection(String connectionName) .. {  
	// 1. getHost(): 返回this.host,默认(初)值为"localhost"
	// 2. getPort(): 
	//      - 指定了this.port(默认初值为-1)  ->  使用指定的port
	//      - 没有指定this.port && AMQP是不带TLS的0-9-1  -> 连接RabbitMQ服务端的5672端口 
	//      - 没有指定this.port && AMQP是带TLS的1.0      -> 连接RabbitMQ服务端的5671端口 
    return newConnection(this.sharedExecutor, Collections.singletonList(new Address(getHost(), getPort())), connectionName);  
}
public Connection newConnection(ExecutorService executor, String connectionName) .. {  
    return newConnection(executor, Collections.singletonList(new Address(getHost(), getPort())), connectionName);  
}
```
-- --
<font color="00E0FF">2）第二部分方法</font>

下面的方法1、2、3都直接使用创建连接的真正逻辑。
```java
// 1
public Connection newConnection(ExecutorService executor, List<Address> addrs, String clientProvidedName)  
        throws IOException, TimeoutException {  
    return newConnection(executor, createAddressResolver(addrs), clientProvidedName);  
}
// 2
public Connection newConnection(ExecutorService executor, AddressResolver addressResolver) throws IOException, TimeoutException {  
    return newConnection(executor, addressResolver, null);  
}
// 3
public Connection newConnection(AddressResolver addressResolver) throws IOException, TimeoutException {  
    return newConnection(this.sharedExecutor, addressResolver, null);  
}
```

# 真正创建连接使用到的参数

1）

```java
protected AddressResolver createAddressResolver(List<Address> addresses) {  
    if (addresses.size() > 1) {  
        return new ListAddressResolver(addresses);  
    } else {
        return new DnsRecordIpAddressResolver(addresses.get(0), isSSL());  
    }  
}
```


