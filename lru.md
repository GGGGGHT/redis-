# lru

## key lru
```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```
每个obj中都有`lru`属性，用来保存对象最后一次被访问的时间
如果要计算某个key的空闲时间，会用服务器的`lruclock`属性记录的时间减去对象的`lru`属性记录的时间得出这个对象空转的时长
可以用`INFO server`查看`lru_clock`该字段 
```log
127.0.0.1:6379> info server
# Server
redis_version:6.2.4
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:cdd247b73d61004a
redis_mode:standalone
os:Linux 5.10.25-linuxkit x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:c11-builtin
gcc_version:8.3.0
process_id:1
process_supervised:no
run_id:1d1b8971bad4f3e59d1e645ddc93f6bde499415c
tcp_port:6379
server_time_usec:1624620106990338
uptime_in_seconds:17879
uptime_in_days:0
hz:10
configured_hz:10
lru_clock:14007370
executable:/data/redis-server
config_file:/usr/local/etc/redis/redis.conf
io_threads_active:0
```

```c
unsigned int getLRUClock(void) {
    return (mstime()/LRU_CLOCK_RESOLUTION) & LRU_CLOCK_MAX;
}
```
`mstime`函数获得到毫秒的时间
