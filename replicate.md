# replicate 

<pre>
-----------         ----------- 
| master  |  复制   | slave    |
-----------         -----------
</pre>

```c
void slaveofCommand(redisClient *c) {
    if (!strcasecmp(c->argv[1]->ptr,"no") &&
        !strcasecmp(c->argv[2]->ptr,"one")) {
        if (server.masterhost) {
            replicationUnsetMaster();
            sds client = catClientInfoString(sdsempty(),c);
            redisLog(REDIS_NOTICE,
                "MASTER MODE enabled (user request from '%s')",client);
            sdsfree(client);
        }
    } else {
        long port;

        if ((getLongFromObjectOrReply(c, c->argv[2], &port, NULL) != REDIS_OK))
            return;

        /* Check if we are already attached to the specified slave */
        if (server.masterhost && !strcasecmp(server.masterhost,c->argv[1]->ptr)
            && server.masterport == port) {
            redisLog(REDIS_NOTICE,"SLAVE OF would result into synchronization with the master we are already connected with. No operation performed.");
            addReplySds(c,sdsnew("+OK Already connected to specified master\r\n"));
            return;
        }
        /* There was no previous master or the user specified a different one,
         * we can continue. */
        replicationSetMaster(c->argv[1]->ptr, port);
        sds client = catClientInfoString(sdsempty(),c);
        redisLog(REDIS_NOTICE,"SLAVE OF %s:%d enabled (user request from '%s')",
            server.masterhost, server.masterport, client);
        sdsfree(client);
    }
    addReply(c,shared.ok);
}
``` 

当使用`SLAVEOF no one`命令后 会取消复制 如果当前的服务端是`master`,则不做任何处理.如果当前的服务端是`slave`则会将该服务端提升为`master` `server.masterhost = NULL`,释放客户端

## 部分复制 

部分重同步功能由三个部分构成 
1. 主服务器的`复制偏移量(replication offset)` 和 从服务器的复制偏移量
2. 主服务器的复制`积压缓冲区(replication backlog)`
3. 服务器运行的ID

### 复制偏移量
执行复制的双方-主从服务器都会分别维护一个复制偏移量: 
 - 主服务器每次向从服务器传播N个字节的数据时,就将自己的复制偏移量的值加上N
 - 从服务器每次收到主服务器传播来的N个字节的数据时,就将自己的复制偏移量的值加上N

拥有相同偏移量的主服务器和三个从服务器
![拥有相同偏移量的主服务器和三个从服务器](https://res.weread.qq.com/wrepub/epub_622000_229)
这服务器向三个从服务器传播长度为33字节的数据后,主从服务器的偏移量都将更新.更新偏移量之后的主从服务器
![更新偏移量之后的主从服务器](https://res.weread.qq.com/wrepub/epub_622000_230)
