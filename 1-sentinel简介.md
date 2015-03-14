##sentinel简介
-------------

### **sentinel是什么**
---------------------

sentinel是redis官方自2.8版本以来的一个auto failover的方案，
auto failover就是master down掉了，自动在其slave里面选一个出来作为新的master,
目标当然是希望能够让redis server能够提供接近于zero downtime的数据服务。

sentinel的启动方式是redis-sentinel /path/to/sentinel.conf或者redis-server /path/to/sentinel.conf --sentinel。

vagrant@vagrant ~/d/redis-2.8.19> make V=s -n | grep "redis-server"

cc -g -ggdb -rdynamic -o redis-server adlist.o ae.o anet.o dict.o redis.o sds.o zmalloc.o lzf_c.o lzf_d.o pqsort.o zipmap.o sha1.o ziplist.o release.o networking.o util.o object.o db.o replication.o rdb.o t_string.o t_list.o t_set.o t_zset.o t_hash.o config.o aof.o pubsub.o multi.o debug.o sort.o intset.o syncio.o migrate.o endianconv.o slowlog.o scripting.o bio.o rio.o rand.o memtest.o crc64.o bitops.o sentinel.o notify.o setproctitle.o hyperloglog.o latency.o sparkline.o ../deps/hiredis/libhiredis.a ../deps/lua/src/liblua.a -lm -pthread ../deps/jemalloc/lib/libjemalloc.a -ldl

install redis-server redis-sentinel

vagrant@vagrant ~/d/redis-2.8.19> which install

/usr/bin/install

可以看出来redis-sentinel和redis-server实际上是同一个binary文件，
只是根据配置不同，会按照不同的方式启动。

```
/* src/redis.c */
1831     /* Create the serverCron() time event, that's our main way to process
1832      * background operations. */
1833     if(aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
1834         redisPanic("Can't create the serverCron time event.");
1835         exit(1);
1836     }

1063 int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
1242     /* Run the Sentinel timer if we are in sentinel mode. */
1243     run_with_period(100) {
1244         if (server.sentinel_mode) sentinelTimer();
1245     }

3542 /* Returns 1 if there is --sentinel among the arguments or if
3543  * argv[0] is exactly "redis-sentinel". */
3544 int checkForSentinelMode(int argc, char **argv) {
3545     int j;
3546
3547     if (strstr(argv[0],"redis-sentinel") != NULL) return 1;
3548     for (j = 1; j < argc; j++)
3549         if (!strcmp(argv[j],"--sentinel")) return 1;
3550     return 0;
3551 }
3688 int main(int argc, char **argv) {
3727     server.sentinel_mode = checkForSentinelMode(argc,argv);
```

sentinelTimer就是整个sentinel逻辑的入口，随着redis server的backgroud任务定期执行。
看sentinel的代码也很容易，几乎所有的sentinel相关逻辑全在src/sentinel.c里面。
sentinelTimer是src/sentinel.c的最后一个func。

```
/* src/sentinel.c */
4010 void sentinelTimer(void) {
```

### **sentinel的用途**
---------------------

sentinel会负责monitor你指定的master，并且自动发现该master的slave以及其他也在monitor该master的sentinel。
sentinel是p2p结构，所以大概的拓扑结构就是一组sentinel instances监控一批redis instances。

**值得注意的是，sentinel知晓由我们指定的各个master之后，会通过两种不同的方式去分别发现
这些属于这些master的slave和一个master下除了当前sentinel之外的其他也在同时monitor该master的其他sentinel，
但是并不会通过与其他sentinel之间的渠道或者更多的其他的渠道去发现更多的master，
不可否认的是，其他sentinel会pubsub自己监控的所有masters的消息给当前sentinel，
但是当前sentinel收到这些消息后，发现遇到了未知的master，会直接忽略相关消息, 或者回复没有建设性的意见。
也就是说master信息是不会通过广播机制传递的, sentinel monitor的master集合并不会自己增长。

如果sentinel从收集来的信息判断master表现得有异常，sentinel就会自动从该master的slave中选一个提升为新的master，
一次failover是由这组sentinel中的一个来主导的，行动之前需要获得组内大多数sentinel的认同。

另外，sentinel是集群部署的，一个sentinel可以说是基本不能体现出sentinel的设计，
集群中的这些sentinel是对等的,即每个sentinel的担当是一样的。
也正是由于这点，sentinel允许大多数都还健在的情况下，down掉一部分节点。

### **sentinel最重要的数据结构**
--------------------------------

```
/* src/sentinel.c */
196 /* Main state. */
197 struct sentinelState {
198     uint64_t current_epoch;     /* Current epoch. */
199     dict *masters;      /* Dictionary of master sentinelRedisInstances.
200                            Key is the instance name, value is the
201                            sentinelRedisInstance structure pointer. */
211 } sentinel;
```

一个redis sentinel进程中只initialize一个这样的global struct，redis本身又不是多线程的。

可以看到这个全局唯一的sentinel struct里面有一个保存了
包含管辖下的所有master struct的pointer的dict，key就是我们指定的master的name，value就是ptr。

```
/* src/sentinel.c */
118 typedef struct sentinelRedisInstance {
119     int flags;      /* See SRI_... defines */
120     char *name;     /* Master name from the point of view of this sentinel. */
121     char *runid;    /* run ID of this instance. */
123     sentinelAddr *addr; /* Master host. */
124     redisAsyncContext *cc; /* Hiredis context for commands. */
125     redisAsyncContext *pc; /* Hiredis context for Pub / Sub. */
159     /* Master specific. */
160     dict *sentinels;    /* Other sentinels monitoring the same master. */
161     dict *slaves;       /* Slaves for this master instance. */
166     /* Slave specific. */
170     struct sentinelRedisInstance *master; /* Master instance if it's slave. */
194 } sentinelRedisInstance;

896 sentinelRedisInstance *createSentinelRedisInstance(char *name, int flags, char *hostname, int port, int quorum, sentinelRedisInstance *master) {
919     if (flags & SRI_MASTER) table = sentinel.masters;
920     else if (flags & SRI_SLAVE) table = master->slaves;
921     else if (flags & SRI_SENTINEL) table = master->sentinels;
992     /* Add into the right table. */
993     dictAdd(table, ri->name, ri);
```

可以看出来，sentinelRedisInstance struct可以表达master、slave、sentinel三种角色，
在上述提到的那个global的sentinel的struct里，masters那个dict里每一个value都指向
这样一个master role的sentinelRedisInstance struct的pointer，
而这样一个master role的sentinelRedisInstance struct里又有dict *sentinels和dict *slaves
这样两个dict，slaves这个dict的value包含该master的所有slaves的指针，sentinels这个
dict里包含有moniter该master的所有other sentinels的指针。而这些指针进一步指向
slave role或者sentinel role的sentinelRedisInstance struct.
而对于每个slave role或者sentinel role的sentinelRedisInstance struct里的*master又是
所属的那个master sentinelRedisInstance struct的指针。

- 去填充这个sentinel.masters这个dict的地方在列举如下：

```
/* src/sentinel.c */
2628 void sentinelCommand(redisClient *c) {
2739     } else if (!strcasecmp(c->argv[1]->ptr,"monitor")) {
2758         /* Parameters are valid. Try to create the master instance. */
2759         ri = createSentinelRedisInstance(c->argv[2]->ptr,SRI_MASTER,
2760                 c->argv[3]->ptr,port,quorum,NULL);

1333 /* ============================ Config handling ============================= */
1334 char *sentinelHandleConfiguration(char **argv, int argc) {
1337     if (!strcasecmp(argv[0],"monitor") && argc == 5) {
1338         /* monitor <name> <host> <port> <quorum> */
1339         int quorum = atoi(argv[4]);
1340
1341         if (quorum <= 0) return "Quorum must be 1 or greater.";
1342         if (createSentinelRedisInstance(argv[1],SRI_MASTER,argv[2],
1343                                         atoi(argv[3]),quorum,NULL) == NULL)
```

- 填充master->slaves这个dict的地方列举如下：
    
```
/* src/sentinel.c */
1226 int sentinelResetMasterAndChangeAddress(sentinelRedisInstance *master, char *ip, int port) {
1265     /* Add slaves back. */
1266     for (j = 0; j < numslaves; j++) {
1267         sentinelRedisInstance *slave;
1268
1269         slave = createSentinelRedisInstance(NULL,SRI_SLAVE,slaves[j]->ip,
1270                     slaves[j]->port, master->quorum, master);

1333 /* ============================ Config handling ============================= */
1334 char *sentinelHandleConfiguration(char **argv, int argc) {
1411     } else if (!strcasecmp(argv[0],"known-slave") && argc == 4) {
1412         sentinelRedisInstance *slave;
1413
1414         /* known-slave <name> <ip> <port> */
1415         ri = sentinelGetMasterByName(argv[1]);
1416         if (!ri) return "No such master with specified name.";
1417         if ((slave = createSentinelRedisInstance(NULL,SRI_SLAVE,argv[2],
1418                     atoi(argv[3]), ri->quorum, ri)) == NULL)

1789 /* Process the INFO output from masters. */
1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
1850             /* Check if we already have this slave into our table,
1851              * otherwise add it. */
1852             if (sentinelRedisInstanceLookupSlave(ri,ip,atoi(port)) == NULL) {
1853                 if ((slave = createSentinelRedisInstance(NULL,SRI_SLAVE,ip,
1854                             atoi(port), ri->quorum, ri)) != NULL)

2378     if ((ri->flags & SRI_SENTINEL) == 0 &&
2379         (ri->info_refresh == 0 ||
2380         (now - ri->info_refresh) > info_period))
2381     {
2382         /* Send INFO to masters and slaves, not sentinels. */
2383         retval = redisAsyncCommand(ri->cc,
2384             sentinelInfoReplyCallback, NULL, "INFO");
2385         if (retval == REDIS_OK) ri->pending_commands++;

2038 void sentinelInfoReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {
2047     if (r->type == REDIS_REPLY_STRING) {
2048         sentinelRefreshInstanceInfo(ri,r->str);
2049     }
2050 }
```
    
- 填充master->sentinels这个dict的地方列举如下：
    
```
/* src/sentinel.c */
1333 /* ============================ Config handling ============================= */
1334 char *sentinelHandleConfiguration(char **argv, int argc) {
1422     } else if (!strcasecmp(argv[0],"known-sentinel") &&
1423                (argc == 4 || argc == 5)) {
1424         sentinelRedisInstance *si;
1425
1426         /* known-sentinel <name> <ip> <port> [runid] */
1427         ri = sentinelGetMasterByName(argv[1]);
1428         if (!ri) return "No such master with specified name.";
1429         if ((si = createSentinelRedisInstance(NULL,SRI_SENTINEL,argv[2],
1430                     atoi(argv[3]), ri->quorum, ri)) == NULL)

2119  * If the master name specified in the message is not known, the message is
2120  * discarded. */
2121 void sentinelProcessHelloMessage(char *hello, int hello_len) {
2128     sentinelRedisInstance *si, *master;
2129
2130     if (numtokens == 8) {
2131         /* Obtain a reference to the master this hello message is about */
2132         master = sentinelGetMasterByName(token[4]);
2155             /* Add the new sentinel. */
2156             si = createSentinelRedisInstance(NULL,SRI_SENTINEL,
2157                             token[0],port,master->quorum,master);
```
有关sentinelProcessHelloMessage的细节后续章节会详细解释。
    
struct sentinelRedisInstance那个代码段有一些注释有错误，
    
```
/* src/sentinel.c */
118 typedef struct sentinelRedisInstance {
120     char *name;     /* Master name from the point of view of this sentinel. */
123     sentinelAddr *addr; /* Master host. */
166     /* Slave specific. */
170     struct sentinelRedisInstance *master; /* Master instance if it's slave. */
194 } sentinelRedisInstance;
```

- *name，并不是slave或者sentinel的createSentinelRedisInstance struct里都存的相应master的name，slave和sentinel有自己的直观的name。
      
```
/* src/sentinel.c */
909     /* For slaves and sentinel we use ip:port as name. */
910     if (flags & (SRI_SLAVE|SRI_SENTINEL)) {
911         anetFormatAddr(slavename, sizeof(slavename), hostname, port);
912         name = slavename;
913     }
```
```
592 /* Format an IP,port pair into something easy to parse. If IP is IPv6
593  * (matches for ":"), the ip is surrounded by []. IP and port are just
594  * separated by colons. This the standard to display addresses within Redis. */
595 int anetFormatAddr(char *buf, size_t buf_len, char *ip, int port) {
596     return snprintf(buf,buf_len, strchr(ip,':') ?
597            "[%s]:%d" : "%s:%d", ip, port);
598 }
```
      
- *addr，每个sentinelRedisInstance都有，不只是master有。
      
- *master，slave或者senitinel的sentinelRedisInstance都有,而不是只有slave有。
      
```
/* src/sentinel.c */
896 sentinelRedisInstance *createSentinelRedisInstance(char *name, int flags, char *hostname, int port, int quorum, sentinelRedisInstance *master) {
902     redisAssert(flags & (SRI_MASTER|SRI_SLAVE|SRI_SENTINEL));
903     redisAssert((flags & SRI_MASTER) || master != NULL);
969     ri->master = master;
```
      
### **sentinel配置概览**
-----------------------

- sentinel monitor mymaster 127.0.0.1 6379 2
    
    sentinel down-after-milliseconds mymaster 60000 
    
    sentinel failover-timeout mymaster 180000 
    
    sentinel parallel-syncs mymaster 1 
    
- sentinel monitor resque 192.168.1.3 6380 4 
    
    sentinel down-after-milliseconds resque 10000
    
    sentinel failover-timeout resque 180000
    
    sentinel parallel-syncs resque 5
    
    以上是两组sentinel配置的实例。没什么特别的，如果是在config中指定这种方式的话，
    请保证sentinel monitor mymaster写在其他该master的配置前面。
    
- monitor那一行，后面的几个参数是master_name, ip, port, quorum.
    
- master_name: 就是给master指定的一个唯一的name，唯一有两个意思，一是两个不同的master在这个sentinel不能用同样的名字，二是一组sentinel的两个sentinel之间对同一个master得用同样的名字，才能方便他们沟通（要是不一样，这个我还没试过）。与其说是指定给master的name，不如说是指定给一个master和他的所有slaves这样一组redis instance的name，指定的name并不会长期和一个ip port的redis instance绑定到一起，会failover嘛！。但是只有表达master的时候会直接用这个名字，比如
    
    master failover-test-bucket20 192.168.31.102 6399
    
    表达该master的slave和监控该master的所有sentinel时（sentinel的struct有一种隶属于master的struct的关系），则会用（只是举例，并没有用一个master举例）
    
    slave 192.168.31.100:6413 192.168.31.100 6413 @ failover-test-bucket48 192.168.31.101 6413
    
    sentinel 192.168.31.100:16379 192.168.31.100 16379 @ failover-test-bucket80 192.168.31.100 6429
    
    这样的表达方式，slave和sentinel直接的name是"%s:%s" % (ip, port)，但是注意"@"字符后面会列出来他所属的master的信息。
    
- quorum：这个数字是配置给投票时统计大多数用的（这个说法很模糊，后续章节会详细解释）
    
```
/* src/sentinel.c */
3335     voters = dictSize(master->sentinels)+1; /* All the other sentinels and me. */
3359         if (votes > max_votes) {
3360             max_votes = votes;
3361             winner = dictGetKey(de);
3362         }
3383     voters_quorum = voters/2+1;
3384     if (winner && (max_votes < voters_quorum || max_votes < master->quorum))
3385         winner = NULL;
```
    
- down-after-milliseconds这一行，是指所有类型的实例进入sdown之前，需要down-after-milliseconds这么长时间没有响应，才能够判定。我们之前测试用的是3.1s，但是在网络异常或者sentinel本身负荷很高的情况下，3.1s其实少了一些，后续可能会增加到10s左右。
    
- failover-timeout这一行是指failover超过这么长时间就会被判定为超时（不准确，后续章节会解释）。
    
- parallel-syncs这一行跟我们关系不大，是用来failover之后限制同时reconf slave of 新的master的slave的个数的，而我们的用法是每个master只有一个slave。
    
```
/* src/sentinel.c */
3811     di = dictGetIterator(master->slaves);
3812     while(in_progress < master->parallel_syncs &&
3813           (de = dictNext(di)) != NULL)
3814     {
3815         sentinelRedisInstance *slave = dictGetVal(de);
3839         /* Send SLAVEOF <new master>. */
3840         retval = sentinelSendSlaveOf(slave,
3841                 master->promoted_slave->addr->ip,
3842                 master->promoted_slave->addr->port);
3843         if (retval == REDIS_OK) {
3844             slave->flags |= SRI_RECONF_SENT;
3845             slave->slave_reconf_sent_time = mstime();
3846             sentinelEvent(REDIS_NOTICE,"+slave-reconf-sent",slave,"%@");
3847             in_progress++;
3848         }
```
