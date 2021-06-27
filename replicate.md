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

/* This function handles the PSYNC command from the point of view of a
 * master receiving a request for partial resynchronization.
 *
 * On success return REDIS_OK, otherwise REDIS_ERR is returned and we proceed
 * with the usual full resync. */
int masterTryPartialResynchronization(redisClient *c) {
    long long psync_offset, psync_len;
    char *master_runid = c->argv[1]->ptr;
    char buf[128];
    int buflen;

    /* Is the runid of this master the same advertised by the wannabe slave
     * via PSYNC? If runid changed this master is a different instance and
     * there is no way to continue. */
    if (strcasecmp(master_runid, server.runid)) {
        /* Run id "?" is used by slaves that want to force a full resync. */
        if (master_runid[0] != '?') {
            redisLog(REDIS_NOTICE,"Partial resynchronization not accepted: "
                "Runid mismatch (Client asked for '%s', I'm '%s')",
                master_runid, server.runid);
        } else {
            redisLog(REDIS_NOTICE,"Full resync requested by slave %s",
                replicationGetSlaveName(c));
        }
        goto need_full_resync;
    }

    /* We still have the data our slave is asking for? */
    if (getLongLongFromObjectOrReply(c,c->argv[2],&psync_offset,NULL) !=
       REDIS_OK) goto need_full_resync;
    if (!server.repl_backlog ||
        psync_offset < server.repl_backlog_off ||
        psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen))
    {
        redisLog(REDIS_NOTICE,
            "Unable to partial resync with slave %s for lack of backlog (Slave request was: %lld).", replicationGetSlaveName(c), psync_offset);
        if (psync_offset > server.master_repl_offset) {
            redisLog(REDIS_WARNING,
                "Warning: slave %s tried to PSYNC with an offset that is greater than the master replication offset.", replicationGetSlaveName(c));
        }
        goto need_full_resync;
    }

    /* If we reached this point, we are able to perform a partial resync:
     * 1) Set client state to make it a slave.
     * 2) Inform the client we can continue with +CONTINUE
     * 3) Send the backlog data (from the offset to the end) to the slave. */
    c->flags |= REDIS_SLAVE;
    c->replstate = REDIS_REPL_ONLINE;
    c->repl_ack_time = server.unixtime;
    c->repl_put_online_on_ack = 0;
    listAddNodeTail(server.slaves,c);
    /* We can't use the connection buffers since they are used to accumulate
     * new commands at this stage. But we are sure the socket send buffer is
     * empty so this write will never fail actually. */
    buflen = snprintf(buf,sizeof(buf),"+CONTINUE\r\n");
    if (write(c->fd,buf,buflen) != buflen) {
        freeClientAsync(c);
        return REDIS_OK;
    }
    psync_len = addReplyReplicationBacklog(c,psync_offset);
    redisLog(REDIS_NOTICE,
        "Partial resynchronization request from %s accepted. Sending %lld bytes of backlog starting from offset %lld.",
            replicationGetSlaveName(c),
            psync_len, psync_offset);
    /* Note that we don't need to set the selected DB at server.slaveseldb
     * to -1 to force the master to emit SELECT, since the slave already
     * has this state from the previous connection with the master. */

    refreshGoodSlavesCount();
    return REDIS_OK; /* The caller can return, no full resync needed. */

need_full_resync:
    /* We need a full resync for some reason... Note that we can't
     * reply to PSYNC right now if a full SYNC is needed. The reply
     * must include the master offset at the time the RDB file we transfer
     * is generated, so we need to delay the reply to that moment. */
    return REDIS_ERR;
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

拥有相同偏移量的主服务器和三个从服务器<br/>
![拥有相同偏移量的主服务器和三个从服务器](https://res.weread.qq.com/wrepub/epub_622000_229)<br/>
当主服务器向三个从服务器传播长度为33字节的数据后,主从服务器的偏移量都将更新.更新偏移量之后的主从服务器<br/>
![更新偏移量之后的主从服务器](https://res.weread.qq.com/wrepub/epub_622000_230)<br/>


### 复制积压缓冲区
复制积压缓冲区是由主服务器维护的一个固定长度先进先出的队列

当主服务器进行命令传播时,它不仅会将写命令发送给所有的从服务器,还会将写命令入队到复制积压区里面<br/>
![](https://res.weread.qq.com/wrepub/epub_622000_232)<br/>

因此,主服务器的缓冲区里会保存着一部分最近传播的写命令,并且复制积压缓冲区会为队列中每个字节记录相应的复制偏移量<br/>
![](https://res.weread.qq.com/wrepub/epub_622000_233)<br/>

当从服务器连接上主服务器时,从服务器会通过`PSYNC`命令来将自己的偏移量`offset`发送给主服务器,主服务器会根据这个偏移量来决定对从服务器执行全量同步或部分同步
 - 如果offset偏移量之后的数据仍然存在于复制积压缓冲区里,那么主服务器将对从服务器执行部分重同步操作
 - 如果offset偏移量之后的数据已经不存在于复制积压缓冲区里,那么主服务器将对从服务器执行完整重同步操作

### 服务器运行ID

除了复制偏移量和复制积压缓冲区,实现部分重同步还需要用到服务器运行ID
每个Redis服务器,不论主从服务器,都会有自己的运行Id 在服务器启动时由`initServerConfig()`函数给赋值 

当从服务器断线并重新连上一个及服务器时,从服务器将向当前连接的主服务器发送之前保存的运行ID
   - 如果从服务器保存的运行ID和当前连接的主服务器运行ID相同,那么主服务器可以<b>`尝试`</b>执行`部分重同步`操作,也需要遵守部分重同步的规则 
   - 如果从服务器保存的运行ID与当前连接的主服务器运行ID不同,那么直接执行`完整同步`操作
