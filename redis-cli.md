# redis客户端

```c
typedef struct client {
    uint64_t id;            /* Client incremental unique ID. */
    connection *conn;
    int resp;               /* RESP protocol version. Can be 2 or 3. */
    redisDb *db;            /* Pointer to currently SELECTed DB. */
    robj *name;             /* As set by CLIENT SETNAME. */
    sds querybuf;           /* Buffer we use to accumulate client queries. */
    size_t qb_pos;          /* The position we have read in querybuf. */
    sds pending_querybuf;   /* If this client is flagged as master, this buffer
                               represents the yet not applied portion of the
                               replication stream that we are receiving from
                               the master. */
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size. */
    int argc;               /* Num of arguments of current command. */
    robj **argv;            /* Arguments of current command. */
    size_t argv_len_sum;    /* Sum of lengths of objects in argv list. */
    struct redisCommand *cmd, *lastcmd;  /* Last command executed. */
    user *user;             /* User associated with this connection. If the
                               user is set to NULL the connection can do
                               anything (admin). */
    int reqtype;            /* Request protocol type: PROTO_REQ_* */
    int multibulklen;       /* Number of multi bulk arguments left to read. */
    long bulklen;           /* Length of bulk argument in multi bulk request. */
    list *reply;            /* List of reply objects to send to the client. */
    unsigned long long reply_bytes; /* Tot bytes of objects in reply list. */
    size_t sentlen;         /* Amount of bytes already sent in the current
                               buffer or object being sent. */
    time_t ctime;           /* Client creation time. */
    time_t lastinteraction; /* Time of the last interaction, used for timeout */
    time_t obuf_soft_limit_reached_time;
    uint64_t flags;         /* Client flags: CLIENT_* macros. */
    int authenticated;      /* Needed when the default user requires auth. */
    int replstate;          /* Replication state if this is a slave. */
    int repl_put_online_on_ack; /* Install slave write handler on first ACK. */
    int repldbfd;           /* Replication DB file descriptor. */
    off_t repldboff;        /* Replication DB file offset. */
    off_t repldbsize;       /* Replication DB file size. */
    sds replpreamble;       /* Replication DB preamble. */
    long long read_reploff; /* Read replication offset if this is a master. */
    long long reploff;      /* Applied replication offset if this is a master. */
    long long repl_ack_off; /* Replication ack offset, if this is a slave. */
    long long repl_ack_time;/* Replication ack time, if this is a slave. */
    long long psync_initial_offset; /* FULLRESYNC reply offset other slaves
                                       copying this slave output buffer
                                       should use. */
    char replid[CONFIG_RUN_ID_SIZE+1]; /* Master replication ID (if master). */
    int slave_listening_port; /* As configured with: REPLCONF listening-port */
    char slave_ip[NET_IP_STR_LEN]; /* Optionally given by REPLCONF ip-address */
    int slave_capa;         /* Slave capabilities: SLAVE_CAPA_* bitwise OR. */
    multiState mstate;      /* MULTI/EXEC state */
    int btype;              /* Type of blocking op if CLIENT_BLOCKED. */
    blockingState bpop;     /* blocking state */
    long long woff;         /* Last write global replication offset. */
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */
    dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */
    list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) */
    sds peerid;             /* Cached peer ID. */
    listNode *client_list_node; /* list node in client list */
    RedisModuleUserChangedFunc auth_callback; /* Module callback to execute
                                               * when the authenticated user
                                               * changes. */
    void *auth_callback_privdata; /* Private data that is passed when the auth
                                   * changed callback is executed. Opaque for
                                   * Redis Core. */
    void *auth_module;      /* The module that owns the callback, which is used
                             * to disconnect the client if the module is
                             * unloaded for cleanup. Opaque for Redis Core.*/

    /* If this client is in tracking mode and this field is non zero,
     * invalidation messages for keys fetched by this client will be send to
     * the specified client ID. */
    uint64_t client_tracking_redirection;
    rax *client_tracking_prefixes; /* A dictionary of prefixes we are already
                                      subscribed to in BCAST mode, in the
                                      context of client side caching. */
    /* In clientsCronTrackClientsMemUsage() we track the memory usage of
     * each client and add it to the sum of all the clients of a given type,
     * however we need to remember what was the old contribution of each
     * client, and in which categoty the client was, in order to remove it
     * before adding it the new value. */
    uint64_t client_cron_last_memory_usage;
    int      client_cron_last_memory_type;
    /* Response buffer */
    int bufpos;
    char buf[PROTO_REPLY_CHUNK_BYTES];
} client;
```

客户端中保存着`id` `conn` 其中`conn`保存着`fd`用来记录客户端正在使用的套接字描述符。 如果是`fakeclient`即用来加载`AOF`文件的客户端 则链接为空 fd也为
`querybuf` 这个缓冲区用来保存客户端发送的命令请求
`argc`表示当前命令参数的个数
`argv`表示当前命令的参数

执行`SET key value`命令时client的argc argv的内存布局
<pre>
-------------
|redisclient|
-------------
|...        |
-------------
|argc   3   |
-------------               -------------------------------------
|argv       |    ------->   | argv[0]   |argv[1]   | argv[2]     |
-------------               --------------------------------------
|...        |               | SET       | key      | value       | 
-------------               --------------------------------------
</pre>

`redisCommand *cmd, *lastcmd` 当服务器从协议内容中分析并得出argv的属性和argc属性的值后，服务器将根据argv[0]的值，在命令表中查询命令所对应的命令处理函数 
cmd即保存着该命令的处理函数 
![查找命令处理函数的流程](https://res.weread.qq.com/wrepub/epub_622000_202)
