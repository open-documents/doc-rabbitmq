
```java
public void handleConfirm(PendingConfirm pendingConfirm, boolean ack) {  
    if (this.confirmCallback != null) {  
        this.confirmCallback.confirm(
	        pendingConfirm.getCorrelationData(), 
	        ack, 
	        pendingConfirm.getCause()
	    ); // NOSONAR never null  
    }  
}
```

```java
public void handleReturn(Return returned) {  
    ReturnsCallback callback = this.returnsCallback;  
    if (callback == null) {  
	    // 1. RETURN_CORRELATION_KEY = "spring_request_return_correlation"
        Object messageTagHeader = returned.getProperties().getHeaders().remove(RETURN_CORRELATION_KEY);  
        if (messageTagHeader != null) {  
            String messageTag = messageTagHeader.toString();  
            final PendingReply pendingReply = this.replyHolder.get(messageTag);  
            if (pendingReply != null) {  
	            callback = (returnedMsg) ->  
                    pendingReply.returned(new AmqpMessageReturnedException("Message returned", returnedMsg));  
            }  
            else{
	            ... // log
            }
        }  
	    else{
		    ... // log
	    }
    }  
    if (callback != null) {  
	    // 1. RETURN_LISTENER_CORRELATION_KEY = "spring_listener_return_correlation"
        returned.getProperties().getHeaders().remove(PublisherCallbackChannel.RETURN_LISTENER_CORRELATION_KEY);  
        MessageProperties messageProperties = this.messagePropertiesConverter.toMessageProperties(  
            returned.getProperties(), null, this.encoding);  
        Message returnedMessage = new Message(returned.getBody(), messageProperties);  
        callback.returnedMessage(new ReturnedMessage(returnedMessage, returned.getReplyCode(),  
            returned.getReplyText(), returned.getExchange(), returned.getRoutingKey()));  
    }  
}
```

```java
public void revoke(Channel channel) {  
    this.publisherConfirmChannels.remove(channel);  
	... // log
}
```