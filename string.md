checkStringLength(client *c, long long size) 校验字符串长度 如果长度大于`proto_max_bulk_len`则报错 `proto_max_bulk_len`默认大小为512m 最小为1kb 小于1kb服务无法启动

当`proto_max_bulk_len`设置为`1kb`时，向redis中写入大于1k的数据后会出现 

```
java.lang.IllegalStateException: Failed to execute CommandLineRunner
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:798) ~[spring-boot-2.5.0.jar:2.5.0]
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:779) ~[spring-boot-2.5.0.jar:2.5.0]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:344) ~[spring-boot-2.5.0.jar:2.5.0]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1336) ~[spring-boot-2.5.0.jar:2.5.0]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1325) ~[spring-boot-2.5.0.jar:2.5.0]
	at com.ggggght.usespringbootstarter.UseSpringBootStarterApplication.main(UseSpringBootStarterApplication.java:70) ~[classes/:na]
Caused by: org.springframework.data.redis.RedisSystemException: Error in execution; nested exception is io.lettuce.core.RedisCommandExecutionException: ERR Protocol error: invalid bulk length
	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:54) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:52) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.lettuce.LettuceExceptionConverter.convert(LettuceExceptionConverter.java:41) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.PassThroughExceptionTranslationStrategy.translate(PassThroughExceptionTranslationStrategy.java:44) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.FallbackExceptionTranslationStrategy.translate(FallbackExceptionTranslationStrategy.java:42) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.lettuce.LettuceConnection.convertLettuceAccessException(LettuceConnection.java:271) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.lettuce.LettuceConnection.await(LettuceConnection.java:1062) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.lettuce.LettuceConnection.lambda$doInvoke$4(LettuceConnection.java:919) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.lettuce.LettuceInvoker$Synchronizer.invoke(LettuceInvoker.java:673) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.lettuce.LettuceInvoker$DefaultSingleInvocationSpec.get(LettuceInvoker.java:589) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.lettuce.LettuceStringCommands.set(LettuceStringCommands.java:96) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.DefaultedRedisConnection.set(DefaultedRedisConnection.java:288) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.connection.DefaultStringRedisConnection.set(DefaultStringRedisConnection.java:986) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.core.DefaultValueOperations$3.inRedis(DefaultValueOperations.java:240) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.core.AbstractOperations$ValueDeserializingRedisCallback.doInRedis(AbstractOperations.java:60) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:222) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:189) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.core.AbstractOperations.execute(AbstractOperations.java:96) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at org.springframework.data.redis.core.DefaultValueOperations.set(DefaultValueOperations.java:236) ~[spring-data-redis-2.5.1.jar:2.5.1]
	at com.ggggght.usespringbootstarter.UseSpringBootStarterApplication.run(UseSpringBootStarterApplication.java:93) ~[classes/:na]
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:795) ~[spring-boot-2.5.0.jar:2.5.0]
	... 5 common frames omitted
Caused by: io.lettuce.core.RedisCommandExecutionException: ERR Protocol error: invalid bulk length
	at io.lettuce.core.internal.ExceptionFactory.createExecutionException(ExceptionFactory.java:137) ~[lettuce-core-6.1.2.RELEASE.jar:6.1.2.RELEASE]
	at io.lettuce.core.internal.ExceptionFactory.createExecutionException(ExceptionFactory.java:110) ~[lettuce-core-6.1.2.RELEASE.jar:6.1.2.RELEASE]
	at io.lettuce.core.protocol.AsyncCommand.completeResult(AsyncCommand.java:120) ~[lettuce-core-6.1.2.RELEASE.jar:6.1.2.RELEASE]
	at io.lettuce.core.protocol.AsyncCommand.complete(AsyncCommand.java:111) ~[lettuce-core-6.1.2.RELEASE.jar:6.1.2.RELEASE]
	at io.lettuce.core.protocol.CommandHandler.complete(CommandHandler.java:746) ~[lettuce-core-6.1.2.RELEASE.jar:6.1.2.RELEASE]
	at io.lettuce.core.protocol.CommandHandler.decode(CommandHandler.java:681) ~[lettuce-core-6.1.2.RELEASE.jar:6.1.2.RELEASE]
	at io.lettuce.core.protocol.CommandHandler.channelRead(CommandHandler.java:598) ~[lettuce-core-6.1.2.RELEASE.jar:6.1.2.RELEASE]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.AbstractChannelHandlerContext.fireChannelRead(AbstractChannelHandlerContext.java:357) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.DefaultChannelPipeline$HeadContext.channelRead(DefaultChannelPipeline.java:1410) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:365) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.DefaultChannelPipeline.fireChannelRead(DefaultChannelPipeline.java:919) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.AbstractNioByteChannel$NioByteUnsafe.read(AbstractNioByteChannel.java:166) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:719) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:655) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:581) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:493) ~[netty-transport-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.util.concurrent.SingleThreadEventExecutor$4.run(SingleThreadEventExecutor.java:989) ~[netty-common-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.util.internal.ThreadExecutorMap$2.run(ThreadExecutorMap.java:74) ~[netty-common-4.1.65.Final.jar:4.1.65.Final]
	at io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30) ~[netty-common-4.1.65.Final.jar:4.1.65.Final]
	at java.base/java.lang.Thread.run(Thread.java:834) ~[na:na]
```

```
127.0.0.1:6379> set msg "hello"
OK
127.0.0.1:6379> object encoding msg
"embstr"
127.0.0.1:6379> set str "123456789012345678901234567890123456789"
OK
127.0.0.1:6379> STRLEN str
(integer) 39
127.0.0.1:6379> object encoding str
"embstr"
127.0.0.1:6379> append str "1"
(integer) 40
127.0.0.1:6379> object encoding str
"raw"
```

```c
/* Create a string object with EMBSTR encoding if it is smaller than
 * REIDS_ENCODING_EMBSTR_SIZE_LIMIT, otherwise the RAW encoding is
 * used.
 *
 * The current limit of 39 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define REDIS_ENCODING_EMBSTR_SIZE_LIMIT 39
robj *createStringObject(char *ptr, size_t len) {
    if (len <= REDIS_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```
当字符串长度小于等于`39`时，使用embstr编码 当长度大于39后 使用raw编码
