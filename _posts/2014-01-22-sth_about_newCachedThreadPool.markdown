---
layout: post
title:  "一个newCachedThreadPool引发的崩溃"
date:   2014-01-22 23:55:26
categories: java
---


记录一个案例:

有一个服务由10来台机器组成一个集群，之前的少量内部通讯使用的是redis 的 pub/sub， 发现部分机器的redis的 sub无法正常工作。

追查日志发现:

```
14:42:37 [redisMessageListenerContainer-1] ERROR o.s.d.r.l.RedisMessageListenerContainer - SubscriptionTask aborted with exception:
java.lang.OutOfMemoryError: unable to create new native thread
at java.lang.Thread.start0(Native Method) [na:1.7.0_19]
at java.lang.Thread.start(Thread.java:691) [na:1.7.0_19]
at org.springframework.core.task.SimpleAsyncTaskExecutor.doExecute(SimpleAsyncTaskExecutor.java:193) ~[super-connect-command.jar:na]
at org.springframework.core.task.SimpleAsyncTaskExecutor.execute(SimpleAsyncTaskExecutor.java:167) ~[super-connect-command.jar:na]
at org.springframework.core.task.SimpleAsyncTaskExecutor.execute(SimpleAsyncTaskExecutor.java:148) ~[super-connect-command.jar:na]
at org.springframework.data.redis.listener.RedisMessageListenerContainer.dispatchMessage(RedisMessageListenerContainer.java:921) ~[super-connect-command.jar:na]
at org.springframework.data.redis.listener.RedisMessageListenerContainer.access$1200(RedisMessageListenerContainer.java:72) ~[super-connect-command.jar:na]
at org.springframework.data.redis.listener.RedisMessageListenerContainer$DispatchMessageListener.onMessage(RedisMessageListenerContainer.java:912) ~[super-connect-command.jar:na]
``` 

刚启动，怎么会缺内存呢，后来发现系统中有个 `ExecutorService executorService = Executors.newCachedThreadPool();`

这是一个长连接系统，这个executorService用于异步处理用户登陆。

而由于系统重启，所有的长连接全部断开重连，使得刚启动时，会有大量的登陆请求，从而使得executorService 瞬间 创建出3000多个线程，而导致启动顺序不靠前的 RedisMessageListenerContainer 启动时无线程可用。。。。。

改成了 ExecutorService executorService = Executors.newFixedThreadPool(128) 暂时修复这个问题。