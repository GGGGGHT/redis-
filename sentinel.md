# sentinel

  哨兵是Redis的高可用性的解决方案,由一个或多个Sentinel实例组成的Sentinel系统可以监视伤科昂然个主服务器,以及这些主服务器的所有从服务器,并在
被监视的主服务器进入下线状态时,自动将下线主服务器下的某个从服务器升级为新的主服务器,然后由新的主服务器代替已下线的主服务器继续处理命令请求,

![](https://res.weread.qq.com/wrepub/epub_622000_249)

![](https://res.weread.qq.com/wrepub/epub_622000_250)

当server1的下线时长超过用户设定的下线时长上限时,Sentinel系统就会对server1执行故障转移操作
  1. 首先Sentinel系统会挑选Server1下的其中一个从服务器,并将这个被选中的从服务器升级为新的主服务器.
  2. 之后Sentinel会向server1下的所有从服务器发送新的复制指令,让他们成为新的主服务器的从服务器,当所有的从服务器都的主服务器时,故障转移操作执行完毕
  3. Sentinel还会继续监视已下线的server1,关在它重新上线时,将它设置为新的主服务器的从服务器.

![故障转移](https://res.weread.qq.com/wrepub/epub_622000_251)
![当server1重新上线后会复制server2](https://res.weread.qq.com/wrepub/epub_622000_252)

## 初始化服务器
```c
 if (server.sentinel_mode) {
        initSentinelConfig();
        initSentinel();
 }
 
void initSentinelConfig(void) {
    server.port = REDIS_SENTINEL_PORT;
}
 
 /* Perform the Sentinel mode initialization. */
void initSentinel(void) {
    unsigned int j;

    /* Remove usual Redis commands from the command table, then just add
     * the SENTINEL command. */
    dictEmpty(server.commands,NULL);
    for (j = 0; j < sizeof(sentinelcmds)/sizeof(sentinelcmds[0]); j++) {
        int retval;
        struct redisCommand *cmd = sentinelcmds+j;

        retval = dictAdd(server.commands, sdsnew(cmd->name), cmd);
        redisAssert(retval == DICT_OK);
    }

    /* Initialize various data structures. */
    sentinel.current_epoch = 0;
    sentinel.masters = dictCreate(&instancesDictType,NULL);
    sentinel.tilt = 0;
    sentinel.tilt_start_time = 0;
    sentinel.previous_time = mstime();
    sentinel.running_scripts = 0;
    sentinel.scripts_queue = listCreate();
    sentinel.announce_ip = NULL;
    sentinel.announce_port = 0;
}

/* This function gets called when the server is in Sentinel mode, started,
 * loaded the configuration, and is ready for normal operations. */
void sentinelIsRunning(void) {
    redisLog(REDIS_WARNING,"Sentinel runid is %s", server.runid);

    if (server.configfile == NULL) {
        redisLog(REDIS_WARNING,
            "Sentinel started without a config file. Exiting...");
        exit(1);
    } else if (access(server.configfile,W_OK) == -1) {
        redisLog(REDIS_WARNING,
            "Sentinel config file %s is not writable: %s. Exiting...",
            server.configfile,strerror(errno));
        exit(1);
    }

    /* We want to generate a +monitor event for every configured master
     * at startup. */
    sentinelGenerateInitialMonitorEvents();
}
 ```
 
 `sentinel`执行的工作和普通Redis服务器执行的工作不同,所以Sentinel的初始化过程不会载入RDB或AOF文件来还原数据库状态
