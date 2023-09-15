
```java
/* ---------------------------------- AMQConnection ---------------------------------- */
public void start() throws IOException, TimeoutException {  
    initializeConsumerWorkService();  
    initializeHeartbeatSender();  
    this._running = true;  
    // Make sure that the first thing we do is to send the header,  
    // which should cause any socket errors to show up for us, rather    // than risking them pop out in the MainLoop    
    AMQChannel.SimpleBlockingRpcContinuation connStartBlocker =  
        new AMQChannel.SimpleBlockingRpcContinuation();  
    // We enqueue an RPC continuation here without sending an RPC  
    // request, since the protocol specifies that after sending    
    // the version negotiation header, the client (connection    
    // initiator) is to wait for a connection.start method to    
    // arrive.    
    _channel0.enqueueRpc(connStartBlocker);  
    try {  
        // The following two lines are akin to AMQChannel's  
        // transmit() method for this pseudo-RPC.       
        _frameHandler.setTimeout(handshakeTimeout);  
        _frameHandler.sendHeader();  
    } catch (IOException ioe) {  
        _frameHandler.close();  
        throw ioe;  
    }  

	
    this._frameHandler.initialize(this);  

    AMQP.Connection.Start connStart;  
    AMQP.Connection.Tune connTune = null;  
    try {  
        connStart =  
                (AMQP.Connection.Start) connStartBlocker.getReply(handshakeTimeout/2).getMethod();  
  
        _serverProperties = Collections.unmodifiableMap(connStart.getServerProperties());  
  
        Version serverVersion =  
                new Version(connStart.getVersionMajor(),  
                                   connStart.getVersionMinor());  
  
        if (!Version.checkVersion(clientVersion, serverVersion)) {  
            throw new ProtocolVersionMismatchException(clientVersion,  
                                                              serverVersion);  
        }  
  
        String[] mechanisms = connStart.getMechanisms().toString().split(" ");  
        SaslMechanism sm = this.saslConfig.getSaslMechanism(mechanisms);  
        if (sm == null) {  
            throw new IOException("No compatible authentication mechanism found - " +  
                                          "server offered [" + connStart.getMechanisms() + "]");  
        }  
  
        String username = credentialsProvider.getUsername();  
        String password = credentialsProvider.getPassword();  
  
        if (credentialsProvider.getTimeBeforeExpiration() != null) {  
            if (this.credentialsRefreshService == null) {  
                throw new IllegalStateException("Credentials can expire, a credentials refresh service should be set");  
            }  
            if (this.credentialsRefreshService.isApproachingExpiration(credentialsProvider.getTimeBeforeExpiration())) {  
                credentialsProvider.refresh();  
                username = credentialsProvider.getUsername();  
                password = credentialsProvider.getPassword();  
            }  
        }  
  
        LongString challenge = null;  
        LongString response = sm.handleChallenge(null, username, password);  
  
        do {  
            Method method = (challenge == null)  
                                    ? new AMQP.Connection.StartOk.Builder()  
                                              .clientProperties(_clientProperties)  
                                              .mechanism(sm.getName())  
                                              .response(response)  
                                              .build()  
                                    : new AMQP.Connection.SecureOk.Builder().response(response).build();  
  
            try {  
                Method serverResponse = _channel0.rpc(method, handshakeTimeout/2).getMethod();  
                if (serverResponse instanceof AMQP.Connection.Tune) {  
                    connTune = (AMQP.Connection.Tune) serverResponse;  
                } else {  
                    challenge = ((AMQP.Connection.Secure) serverResponse).getChallenge();  
                    response = sm.handleChallenge(challenge, username, password);  
                }  
            } catch (ShutdownSignalException e) {  
                Method shutdownMethod = e.getReason();  
                if (shutdownMethod instanceof AMQP.Connection.Close) {  
                    AMQP.Connection.Close shutdownClose = (AMQP.Connection.Close) shutdownMethod;  
                    if (shutdownClose.getReplyCode() == AMQP.ACCESS_REFUSED) {  
                        throw new AuthenticationFailureException(shutdownClose.getReplyText());  
                    }  
                }  
                throw new PossibleAuthenticationFailureException(e);  
            }  
        } while (connTune == null);  
    } catch (TimeoutException te) {  
        _frameHandler.close();  
        throw te;  
    } catch (ShutdownSignalException sse) {  
        _frameHandler.close();  
        throw AMQChannel.wrap(sse);  
    } catch(IOException ioe) {  
        _frameHandler.close();  
        throw ioe;  
    }  
  
    try {  
        int negotiatedChannelMax =  
            negotiateChannelMax(this.requestedChannelMax,  
                                connTune.getChannelMax());  
  
        int channelMax = ConnectionFactory.ensureUnsignedShort(negotiatedChannelMax);  
  
        if (channelMax != negotiatedChannelMax) {  
            LOGGER.warn("Channel max must be between 0 and {}, value has been set to {} instead of {}",  
                    MAX_UNSIGNED_SHORT, channelMax, negotiatedChannelMax);  
        }  
  
        _channelManager = instantiateChannelManager(channelMax, threadFactory);  
  
        int frameMax =  
            negotiatedMaxValue(this.requestedFrameMax,  
                               connTune.getFrameMax());  
        this._frameMax = frameMax;  
  
        int negotiatedHeartbeat =  
            negotiatedMaxValue(this.requestedHeartbeat,  
                               connTune.getHeartbeat());  
  
        int heartbeat = ConnectionFactory.ensureUnsignedShort(negotiatedHeartbeat);  
  
        if (heartbeat != negotiatedHeartbeat) {  
            LOGGER.warn("Heartbeat must be between 0 and {}, value has been set to {} instead of {}",  
                    MAX_UNSIGNED_SHORT, heartbeat, negotiatedHeartbeat);  
        }  
  
        setHeartbeat(heartbeat);  
  
        _channel0.transmit(new AMQP.Connection.TuneOk.Builder()  
                            .channelMax(channelMax)  
                            .frameMax(frameMax)  
                            .heartbeat(heartbeat)  
                          .build());  
        _channel0.exnWrappingRpc(new AMQP.Connection.Open.Builder()  
                                  .virtualHost(_virtualHost)  
                                .build());  
    } catch (IOException ioe) {  
        _heartbeatSender.shutdown();  
        _frameHandler.close();  
        throw ioe;  
    } catch (ShutdownSignalException sse) {  
        _heartbeatSender.shutdown();  
        _frameHandler.close();  
        throw AMQChannel.wrap(sse);  
    }  
  
    if (this.credentialsProvider.getTimeBeforeExpiration() != null) {  
        String registrationId = this.credentialsRefreshService.register(credentialsProvider, () -> {  
            // return false if connection is closed, so refresh service can get rid of this registration  
            if (!isOpen()) {  
                return false;  
            }  
            if (this._inConnectionNegotiation) {  
                // this should not happen  
                return true;  
            }  
            String refreshedPassword = credentialsProvider.getPassword();  
  
            UpdateSecretExtension.UpdateSecret updateSecret = new UpdateSecretExtension.UpdateSecret(  
                    LongStringHelper.asLongString(refreshedPassword), "Refresh scheduled by client"  
            );  
            try {  
                _channel0.rpc(updateSecret);  
            } catch (ShutdownSignalException e) {  
                LOGGER.warn("Error while trying to update secret: {}. Connection has been closed.", e.getMessage());  
                return false;            }  
            return true;  
        });  
  
        addShutdownListener(sse -> this.credentialsRefreshService.unregister(this.credentialsProvider, registrationId));  
    }  
  
    // We can now respond to errors having finished tailoring the connection  
    this._inConnectionNegotiation = false;  
}
```