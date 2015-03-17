## **sentinel failover重点细节**
--------------------------------

### **sentinel与redis, sentinel与sentinel instance之间的交互方式**
------------------------------------------------------------------

其实很简单，这些sentinel redis instance之间唯一的通信方式就是通过tcp通信，
而目前由于sentinel的所有网络通信都是由redisAsyncCommand这个命令异步执行的,
所以只要grep redisAsyncCommand即可list出所有操作。

正如之前提到的，sentinel instance通过config file指定也好，runtime config也好，monitor管辖了很多
所有master instance，对于这些master instance以及这些master的所有slave instance，每个instance建立
了两个tcp连接，一个cc，一个pc。而与此同时，对于每个master instance而言，有other sentine也在
monitor该master instance，sentinel一一与这些other sentinel建立一个cc连接。

先说建立连接时会用到的交互，这部分内容对sentinel与redis，sentinel与sentinel来讲，是通用的。

```
/* src/sentinel.c */
1676 void sentinelSendAuthIfNeeded(sentinelRedisInstance *ri, redisAsyncContext *c) {
1677     char *auth_pass = (ri->flags & SRI_MASTER) ? ri->auth_pass :
1678                                                  ri->master->auth_pass;
1679
1680     if (auth_pass) {
1681         if (redisAsyncCommand(c, sentinelDiscardReplyCallback, NULL, "AUTH %s",
1682             auth_pass) == REDIS_OK) ri->pending_commands++;
1683     }
```

与sentinelSendAuthIfNeeded相关的配置项如下,

```
/* sentinel.conf */
# sentinel auth-pass <master-name> <password>
#
# Set the password to use to authenticate with the master and slaves.
# Useful if there is a password set in the Redis instances to monitor.
#
# Note that the master password is also used for slaves, so it is not
# possible to set a different password in masters and slaves instances
# if you want to be able to monitor these instances with Sentinel.
#
# However you can have Redis instances without the authentication enabled
# mixed with Redis instances requiring the authentication (as long as the
# password set is the same for all the instances requiring the password) as
# the AUTH command will have no effect in Redis instances with authentication
# switched off.
#
# Example:
#
# sentinel auth-pass mymaster MySUPER--secret-0123passw0rd
```

先定义一下bucket这个概念，一个redis master instance以及与他同步的所有redis slave instance这样一组
redis instance称之为一个bucket.

这段配置的含义是，即使这个配置文件只指定了master sentinelRedisInstance的auth passwd，
但是会自动扩散到slave sentinelRedisInstance，主要是为了方便, 为了配合这个做法,才限制如果要和sentinel配合使用，
则同属一个bucket的master和slave role的redis instance必须用相同的auth密码.
也就可以看到*auth_pass = (ri->flags & SRI_MASTER) ? ri->auth_pass : ri->master->auth_pass;就是因为如此.

可以看到此举是针对master或者slave role的sentinelRedisInstance执行的，而sentinelSendAuthIfNeeded在
该sentinelRedisInstance为sentinel role时也会调用，并且此时会走
*auth_pass = (ri->flags & SRI_MASTER) ? ri->auth_pass : ri->master->auth_pass;后半部分的逻辑，那么通过
sentinelSendAuthIfNeeded这个函数给sentinel instance发送ri->master->auth_pass auth信息，有用吗?
会对sentinel instance产生什么影响。在这个问题很简单，因为sentinel instance在启动的时候加载了一个自定义的
响应命令的子集sentinelcmds，这个sentinelcmds list里面根本就没有auth这个cmd，所以，auth命令发送给sentinel instance，
会被直接无视,没有任何影响。后续会介绍sentinelcmds相关逻辑。

```
/* src/sentinel.c */
1686 /* Use CLIENT SETNAME to name the connection in the Redis instance as
1687  * sentinel-<first_8_chars_of_runid>-<connection_type>
1688  * The connection type is "cmd" or "pubsub" as specified by 'type'.
1689  *
1690  * This makes it possible to list all the sentinel instances connected
1691  * to a Redis servewr with CLIENT LIST, grepping for a specific name format. */
1692 void sentinelSetClientName(sentinelRedisInstance *ri, redisAsyncContext *c, char *type) {
1695     snprintf(name,sizeof(name),"sentinel-%.8s-%s",server.runid,type);
1696     if (redisAsyncCommand(c, sentinelDiscardReplyCallback, NULL,
1697         "CLIENT SETNAME %s", name) == REDIS_OK)
```

注释说的很清楚了，CLIENT SETNAME是让在remote redis instance或者sentinel instance按照此命令参数指定的具有规则的名字来命名
这些cc或者pc连接。以便在这些instance上执行CLIENT LIST cmd获取到client list后，可以通过grep相关pattern来筛选过滤.
**TODO, 由于sentinel长时间运行下，可以会产生连接泄露，也许是与某些配置项太小有关系,但是目前不清楚具体原因，希望通过
CLIENT LIST来排查，但是还是上面这个sentinelcmds list子集的问题，需要sentinel同时加载CLIENT LIST,CLIENT SETNAME,
才会让debug成为可能,所以其实现在CLIENT SETNAME cmd发送给sentinel instance，其实是被pass掉的**

然后sentinel对redis或者sentinel instance的ping操作以及sentinelPingReplyCallback中检查到instance处于
BUSY状态时采取SCRIPT KILL操作，这部分内容对sentinel与redis，sentinel与sentinel来讲，也是通用的。

```
/* src/sentinel.c */
2327 int sentinelSendPing(sentinelRedisInstance *ri) {
2328     int retval = redisAsyncCommand(ri->cc,
2329         sentinelPingReplyCallback, NULL, "PING");

2062 void sentinelPingReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {
2084             if (strncmp(r->str,"BUSY",4) == 0 &&
2085                 (ri->flags & SRI_S_DOWN) &&
2086                 !(ri->flags & SRI_SCRIPT_KILL_SENT))
2087             {
2088                 if (redisAsyncCommand(ri->cc,
2089                         sentinelDiscardReplyCallback, NULL,
2090                         "SCRIPT KILL") == REDIS_OK)
```

除了上面提到instance之间的通用的交互方式之外，接下来分开说一下不通用的部分，

先说sentinel与redis instance之间的交互.

sentinel与redis instance之间,

- 先说通过cc进行的,

    - info操作之前也讲过，是通过master或者slave role的sentinelRedisInstance的cc连接进行的。

        ```
        /* src/sentinel.c */
        2344 void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {
        2378     if ((ri->flags & SRI_SENTINEL) == 0 &&
        2379         (ri->info_refresh == 0 ||
        2380         (now - ri->info_refresh) > info_period))
        2381     {
        2382         /* Send INFO to masters and slaves, not sentinels. */
        2383         retval = redisAsyncCommand(ri->cc,
        2384             sentinelInfoReplyCallback, NULL, "INFO");
        ```

    - sentinelSendSlaveOf里有一个transaction，几个相关的命令在里面一并执行,
    这些命令会在master或者slave role的sentinelRedisInstance的cc连接上执行.

        ```
        /* src/sentinel.c */
        3403 int sentinelSendSlaveOf(sentinelRedisInstance *ri, char *host, int port) {
        3426     retval = redisAsyncCommand(ri->cc,
        3427         sentinelDiscardReplyCallback, NULL, "MULTI");
        3428     if (retval == REDIS_ERR) return retval;
        3429     ri->pending_commands++;
        3430
        3431     retval = redisAsyncCommand(ri->cc,
        3432         sentinelDiscardReplyCallback, NULL, "SLAVEOF %s %s", host, portstr);
        3433     if (retval == REDIS_ERR) return retval;
        3434     ri->pending_commands++;
        3435
        3436     retval = redisAsyncCommand(ri->cc,
        3437         sentinelDiscardReplyCallback, NULL, "CONFIG REWRITE");
        3438     if (retval == REDIS_ERR) return retval;
        ```

- 再说通过pc进行的,

    - master或者slave role的sentinelRedisInstance的pc连接(这个连接就是从
    当前sentinel instance连接到remote master或者slave instance)创建之后，
    不可忽略的一个重要操作就是SUBSCRIBE SENTINEL_HELLO_CHANNEL这个频道。

        ```
        /* src/sentinel.c */
        1706 void sentinelReconnectInstance(sentinelRedisInstance *ri) {
        1735     if ((ri->flags & (SRI_MASTER|SRI_SLAVE)) && ri->pc == NULL) {
        1757             retval = redisAsyncCommand(ri->pc,
        1758                 sentinelReceiveHelloMessages, NULL, "SUBSCRIBE %s",
        1759                     SENTINEL_HELLO_CHANNEL);
        ```

    **值得注意的是，可以看到sentinel与sentinel之间并不会直接订阅对方，但是后续会提到的，
    我们配合sentinel的方案中，我们的listener是直接订阅了所有的sentinel instance的，
    即sentinel instance的pubsub的消息来源渠道并不对外开放。怎么做到不开放后续会解释。而是通过
    sentinelEvent方法向外部广播sentinel内部正在发生什么的时候内部使用。**
    关于sentinelEvent方法后续也会详细介绍.

再说sentinel与sentinel instance之间,

- 通过cc进行的,

    - 之前讲到过，sentinel与sentinel instance之间会通过在通向其他sentinel instance的cc连接上执行
      SENTINEL is-master-down-by-addr命令来沟通master的S_DOWN状态，
      并且存储到本地other sentinel sentinelRedisInstance struct的SRI_MASTER_DOWN状态中,供后续统计。

        ```
        /* src/sentinel.c */
        3193 void sentinelAskMasterStateToOtherSentinels(sentinelRedisInstance *master, int flags) {
        3197     di = dictGetIterator(master->sentinels);
        3198     while((de = dictNext(di)) != NULL) {
        3199         sentinelRedisInstance *ri = dictGetVal(de);
        3224         retval = redisAsyncCommand(ri->cc,
        3225                     sentinelReceiveIsMasterDownReply, NULL,
        3226                     "SENTINEL is-master-down-by-addr %s %s %llu %s",
        3227                     master->addr->ip, port,
        3228                     sentinel.current_epoch,
        3229                     (master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ?
        3230                     server.runid : "*");
        ```

至此列出了几乎所有的的sentinel与redis instance之间以及sentinel与sentinel instance之间的交互方式，除了一个例外，
下一章会讲一下，一个重要的但是比较特殊的交互方式, hello msg.可以简单的说，这其实还是一个sentinel与sentinel instance，
sentinel与redis instance之间都会有的交互方式，但是具体交互方式又很不相同。

### **包含hello msg的细节**
---------------------------

先讲一下sentinel instance send hello msg的常规方式,

```
/* src/sentinel.c */
3919 void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {
3923     sentinelSendPeriodicCommands(ri);

2344 void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {
2389     } else if ((now - ri->last_pub_time) > SENTINEL_PUBLISH_PERIOD) {
2390         /* PUBLISH hello messages to all the three kinds of instances. */
2391         sentinelSendHello(ri);
```

```
/* src/redis.c */
1063 int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
1242     /* Run the Sentinel timer if we are in sentinel mode. */
1243     run_with_period(100) {
1244         if (server.sentinel_mode) sentinelTimer();
1245     }
```

可以看到sentinelSendHello的执行是随着sentinelHandleRedisInstance这个sentinelTimer定期执行逻辑执行的.
作用于三种role的sentinelRedisInstance的cc连接上，预计100ms一次。但是有个限制条件就是
距离sentinel三种role的sentinelRedisInstance上次ri->last_pub_time更新已经超过SENTINEL_PUBLISH_PERIOD,
SENTINEL_PUBLISH_PERIOD默认为2s.ri->last_pub_time后续马上会提到。

接下来，详细解释一下sentinelSendHello逻辑，

首先看一下hello msg的格式，

> sentinel_ip,sentinel_port,sentinel_runid,current_epoch,
> master_name,master_ip,master_port,master_config_epoch.

可以看到这个用逗号分隔的msg里含有以下几种信息,

- sentinel_ip,sentinel_port,sentinel_runid这些都是广播关于当前sentinel的信息，让other sentinel发现自己的存在。
注意到sentinel runid信息是间接给vote逻辑用的,但是hello msg跟vote逻辑没有直接关系。

- current_epoch是在global sentinel struct里保存的一个全局epoch信息，后续会详细解释其用途。

- master_name,master_ip,master_port 之前提到过，此send hello msg的逻辑作用于三种role中任意role的sentinelRedisInstance上，
所以此处的master_xx是指当任意role的sentinelRedisInstance对应的master sentinelRedisInstance的ip,port信息。

- master_config_epoch是当前sentinelRedisInstance对应的master sentinelRedisInstance的config_epoch信息，
这个epoch后续也会解释其用途。

有几个重要的问题值得提起，

- 可以看到的是，这个hello msg的各个组成部分实际上是从master sentinelRedisInstance struct中获取的相关config信息,
而这些master sentinelRedisInstance struct实际上是当前sentinel的管辖下的master instance到当前sentinel的env的映射而已,
所以这些信息都是当前sentinel的主观视角的信息而已，保证这些信息的时效性不在此处.
这些master config信息尽可能及时被更新的逻辑后续会提到。

- hello msg是从本地的master slave sentinel三种role的sentinelRedisInstance发起的，
也就是说其实slave sentinel role的sentinelRedisInstance发起的
hello msg其实是同对应的master role的sentinelRedisInstance的hello msg是重复的。
但是注意cc link这个渠道不一样，每个sentinelRedisInstance向外广播的渠道是当前sentinel与这个
sentinelRedisInstance所指向的remote master或slave redis instance或者sentinel instance的之间建立的cc连接。
暂且先不说这些instance对hello msg的处理有何不同，后续会马上提到。

hello msg通过publish cmd不断向外send广播出去，

- 这个广播既发给了master和slave redis instance,
很好理解，通过这些redis instance的pubsub广播渠道曲线到达other sentinel instance，
因为正如上面提到的这一组sentinel中每个sentinel instance都SUBSCRIBE了所有这一组sentinel管辖下
的master和slave instance的SENTINEL_HELLO_CHANNEL channel。

- 同时还直接发给了sentinel instance，
这一点很蹊跷，后续会讲到sentinel instance对通过publish cmd发送hello msg给他的处理方式。

然后我们详细看一下sentinelSendHello的具体逻辑,

```
/* src/sentinel.c */
2250 int sentinelSendHello(sentinelRedisInstance *ri) {
2239 /* Send an "Hello" message via Pub/Sub to the specified 'ri' Redis
2240  * instance in order to broadcast the current configuraiton for this
2241  * master, and to advertise the existence of this Sentinel at the same time.
2242  *
2243  * The message has the following format:
2244  *
2245  * sentinel_ip,sentinel_port,sentinel_runid,current_epoch,
2246  * master_name,master_ip,master_port,master_config_epoch.
2247  *
2248  * Returns REDIS_OK if the PUBLISH was queued correctly, otherwise
2249  * REDIS_ERR is returned. */
2250 int sentinelSendHello(sentinelRedisInstance *ri) {
2251     char ip[REDIS_IP_STR_LEN];
2252     char payload[REDIS_IP_STR_LEN+1024];
2253     int retval;
2254     char *announce_ip;
2255     int announce_port;
2256     sentinelRedisInstance *master = (ri->flags & SRI_MASTER) ? ri : ri->master;
2257     sentinelAddr *master_addr = sentinelGetCurrentMasterAddress(master);
2258
2259     if (ri->flags & SRI_DISCONNECTED) return REDIS_ERR;
2260
2261     /* Use the specified announce address if specified, otherwise try to
2262      * obtain our own IP address. */
2263     if (sentinel.announce_ip) {
2264         announce_ip = sentinel.announce_ip;
2265     } else {
2266         if (anetSockName(ri->cc->c.fd,ip,sizeof(ip),NULL) == -1)
2267             return REDIS_ERR;
2268         announce_ip = ip;
2269     }
2270     announce_port = sentinel.announce_port ?
2271                     sentinel.announce_port : server.port;
2272
2273     /* Format and send the Hello message. */
2274     snprintf(payload,sizeof(payload),
2275         "%s,%d,%s,%llu," /* Info about this sentinel. */
2276         "%s,%s,%d,%llu", /* Info about current master. */
2277         announce_ip, announce_port, server.runid,
2278         (unsigned long long) sentinel.current_epoch,
2279         /* --- */
2280         master->name,master_addr->ip,master_addr->port,
2281         (unsigned long long) master->config_epoch);
2282     retval = redisAsyncCommand(ri->cc,
2283         sentinelPublishReplyCallback, NULL, "PUBLISH %s %s",
2284             SENTINEL_HELLO_CHANNEL,payload);
2285     if (retval != REDIS_OK) return REDIS_ERR;
2286     ri->pending_commands++;
2287     return REDIS_OK;
2288 }
```

- 可以看到如果sentinelRedisInstance处于SRI_DISCONNECTED，则会直接返回REDIS_ERR

- hello msg中sentinel_ip, sentinel_port信息是可以单独从配置文件指定的即announce_host,announce_port。
好处是在docker container的net为bridge mode下，sentinel hello msg机制也可以工作。

- master_xx这些config是从(ri->flags & SRI_MASTER) ? ri : ri->master；这样的sentinelRedisInstance中
通过sentinelGetCurrentMasterAddress获取的。

    sentinelgetcurrentmasteraddress这样一种获取master config的方式值得说一下，

    ```
    /* src/sentinel.c */
    1297 /* Return the current master address, that is, its address or the address
    1298  * of the promoted slave if already operational. */
    1299 sentinelAddr *sentinelGetCurrentMasterAddress(sentinelRedisInstance *master) {
    1300     /* If we are failing over the master, and the state is already
    1301      * SENTINEL_FAILOVER_STATE_RECONF_SLAVES or greater, it means that we
    1302      * already have the new configuration epoch in the master, and the
    1303      * slave acknowledged the configuration switch. Advertise the new
    1304      * address. */
    1305     if ((master->flags & SRI_FAILOVER_IN_PROGRESS) &&
    1306         master->promoted_slave &&
    1307         master->failover_state >= SENTINEL_FAILOVER_STATE_RECONF_SLAVES)
    1308     {
    1309         return master->promoted_slave->addr;
    1310     } else {
    1311         return master->addr;
    1312     }
    1313 }
    ```

    可以看到,

    - 这个master sentinelRedisInstance的flags如果处于SRI_FAILOVER_IN_PROGRESS状态

    - 并且master->promoted_slave为真，

    - 并且master->failover_state >= SENTINEL_FAILOVER_STATE_RECONF_SLAVES, 

    **则表示该promoted_slave所对应的redis instance已经响应了slave of no one的命令摒弃了与old master之间的sync关系,
    此时当前sentinel就开始广播这一虽然是阶段性但确是里程碑性质的成果，
    虽然此时failover还在继续中，但是最重要的一步已经完成.**
    再重提一下sentinelAbortFailover进行的前提条件，

    ```
    /* src/sentinel.c */
    3900 void sentinelAbortFailover(sentinelRedisInstance *ri) {
    3901     redisAssert(ri->flags & SRI_FAILOVER_IN_PROGRESS);
    3902     redisAssert(ri->failover_state <= SENTINEL_FAILOVER_STATE_WAIT_PROMOTION);
    ```

    **可以看到sentinelAbortFailover会redisAssert(ri->failover_state <= SENTINEL_FAILOVER_STATE_WAIT_PROMOTION),而
    SENTINEL_FAILOVER_STATE_WAIT_PROMOTION刚好是SENTINEL_FAILOVER_STATE_RECONF_SLAVES这个状态的前一个状态，到达
    SENTINEL_FAILOVER_STATE_RECONF_SLAVES则表示不能再abort failover,进入sentinelFailoverReconfNextSlave之后该次failover无论
    如何都必须继续完成，所谓必须完成的相关逻辑在sentinelFailoverDetectEnd，即使输出了+failover-end-for-timeout messge，
    该次failover也一定会走+failover-end的逻辑完成,之前将failover流程的时候已经提到过了**

    **可以看到此处就将failover成果通过upgrade config的方式第一时间广播出去，对提高sentinel方案的容错性有很大的好处，
    因为hello msg中master config epoch高的upgrade config一定会获得other sentinel的直接认同
    (除了比较config_epoch之外不需要任何前置确认信息),
    只要有一个sentinel instance将这份高epoch的config持久化下来，这份config就会强制生效了。除非后续有新的config来覆盖它，
    否则redis instance之间一定会达到这个config所定义的拓扑状态，值得注意的是，
    config_epoch的的作用范围以及config_epoch每次变更是局限在一个master的范围内的.**

继续来看sentinel给send hello msg这一PUBLISH async cmd注册的sentinelPublishReplyCallback函数。
同样返回REDIS_ERR在sentinelSendHello表示async cmd根本就没有queued correctly。
可以注意到的是，在sentinelSendHello里并没有直接更新ri->last_pub_time，
更新是在sentinelPublishReplyCallback函数里完成的,
如果reply不为error的情况下才会更新ri->last_pub_time,具体如下,

```
/* src/sentinel.c */
2099 /* This is called when we get the reply about the PUBLISH command we send
2100  * to the master to advertise this sentinel. */
2101 void sentinelPublishReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {
2102     sentinelRedisInstance *ri = c->data;
2103     redisReply *r;
2104     REDIS_NOTUSED(privdata);
2105
2106     if (ri) ri->pending_commands--;
2107     if (!reply || !ri) return;
2108     r = reply;
2109
2110     /* Only update pub_time if we actually published our message. Otherwise
2111      * we'll retry again in 100 milliseconds. */
2112     if (r->type != REDIS_REPLY_ERROR)
2113         ri->last_pub_time = mstime();
2114 }
```

关于ri->last_pub_time，这个参数详细提一下，其限制作用之前已经提过了，
通过now - ri->last_pub_time) > SENTINEL_PUBLISH_PERIOD这个判断来限制调用sentinelSendHello的频率,
而且sentinelSendHello有且仅有那样一个入口。
所以要改变sentinelSendHello的行为，则就只能通过变更ri->last_pub_time来控制。

但是什么情况下更新ri->last_pub_time,上面讲到的只是正常情况下的一种情况,
下面还有一种情况下，为了尽快publish变更出去，
会将当前的ri->last_pub_time减掉SENTINEL_PUBLISH_PERIOD+1这样一个时间间隔，那么
下次循环就会立即执行此publish操作。

具体细节如下，

```
/* src/sentinel.c */
2290 /* Reset last_pub_time in all the instances in the specified dictionary
2291  * in order to force the delivery of an Hello update ASAP. */
2292 void sentinelForceHelloUpdateDictOfRedisInstances(dict *instances) {
2293     dictIterator *di;
2294     dictEntry *de;
2295
2296     di = dictGetSafeIterator(instances);
2297     while((de = dictNext(di)) != NULL) {
2298         sentinelRedisInstance *ri = dictGetVal(de);
2299         if (ri->last_pub_time >= (SENTINEL_PUBLISH_PERIOD+1))
2300             ri->last_pub_time -= (SENTINEL_PUBLISH_PERIOD+1);
2301     }
2302     dictReleaseIterator(di);
2303 }
2304
2305 /* This function forces the delivery of an "Hello" message (see
2306  * sentinelSendHello() top comment for further information) to all the Redis
2307  * and Sentinel instances related to the specified 'master'.
2308  *
2309  * It is technically not needed since we send an update to every instance
2310  * with a period of SENTINEL_PUBLISH_PERIOD milliseconds, however when a
2311  * Sentinel upgrades a configuration it is a good idea to deliever an update
2312  * to the other Sentinels ASAP. */
2313 int sentinelForceHelloUpdateForMaster(sentinelRedisInstance *master) {
2314     if (!(master->flags & SRI_MASTER)) return REDIS_ERR;
2315     if (master->last_pub_time >= (SENTINEL_PUBLISH_PERIOD+1))
2316         master->last_pub_time -= (SENTINEL_PUBLISH_PERIOD+1);
2317     sentinelForceHelloUpdateDictOfRedisInstances(master->sentinels);
2318     sentinelForceHelloUpdateDictOfRedisInstances(master->slaves);
2319     return REDIS_OK;
2320 }
```

可以看到sentinelForceHelloUpdateForMaster会在master sentinelRedisInstance这个struct上执行
该master->last_pub_time减少操作以提前下次send hello msg。

sentinelForceHelloUpdateForMaster的调用时机如下,

```
/* src/sentinel.c */
1789 /* Process the INFO output from masters. */
1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
1944     /* Handle slave -> master role switch. */
1945     if ((ri->flags & SRI_SLAVE) && role == SRI_MASTER) {
1946         /* If this is a promoted slave we can change state to the
1947          * failover state machine. */
1948         if ((ri->flags & SRI_PROMOTED) &&
1949             (ri->master->flags & SRI_FAILOVER_IN_PROGRESS) &&
1950             (ri->master->failover_state ==
1951                 SENTINEL_FAILOVER_STATE_WAIT_PROMOTION))
1952         {
1958             ri->master->config_epoch = ri->master->failover_epoch;
1959             ri->master->failover_state = SENTINEL_FAILOVER_STATE_RECONF_SLAVES;
1960             ri->master->failover_state_change_time = mstime();
1961             sentinelFlushConfig();
1962             sentinelEvent(REDIS_WARNING,"+promoted-slave",ri,"%@");
1963             sentinelEvent(REDIS_WARNING,"+failover-state-reconf-slaves",
1964                 ri->master,"%@");
1965             sentinelCallClientReconfScript(ri->master,SENTINEL_LEADER,
1966                 "start",ri->master->addr,ri->addr);
1967             sentinelForceHelloUpdateForMaster(ri->master);
```

调用时机还是在之前提到的那个关键步骤，failover_state被提升为SENTINEL_FAILOVER_STATE_RECONF_SLAVES
之后立即执行sentinelForceHelloUpdateForMaster，提前下一次send hello msg到下个timer循环，尽快将新的config广播出去。
使得其他sentinel尽快更新自己的config为该upgrade之后的config.

至此关于当前sentinel instance send hello msg以及send hello msg callback已经讲完了.

但是other sentinel instance怎么收到hello msg以及怎么处理hello msg还没有讲，接下来讲一下，对hello msg的响应,

```
/* src/sentinel.c */
1706 void sentinelReconnectInstance(sentinelRedisInstance *ri) {
1756             /* Now we subscribe to the Sentinels "Hello" channel. */
1757             retval = redisAsyncCommand(ri->pc,
1758                 sentinelReceiveHelloMessages, NULL, "SUBSCRIBE %s",
1759                     SENTINEL_HELLO_CHANNEL);
```

- 在SUBSCRIBE master,slave的redis instance的时候,给该channel的pubsub消息注册了一个回调函数sentinelReceiveHelloMessages。
这就是通过pubsub渠道间接获取其他sentinel的hello msg并处理的机制。

    ```
    /* src/sentinel.c */
    2209 /* This is our Pub/Sub callback for the Hello channel. It's useful in order
    2210  * to discover other sentinels attached at the same master. */
    2211 void sentinelReceiveHelloMessages(redisAsyncContext *c, void *reply, void *privdata) {
    2212     sentinelRedisInstance *ri = c->data;
    2213     redisReply *r;
    2214     REDIS_NOTUSED(privdata);
    2215
    2216     if (!reply || !ri) return;
    2217     r = reply;
    2218
    2219     /* Update the last activity in the pubsub channel. Note that since we
    2220      * receive our messages as well this timestamp can be used to detect
    2221      * if the link is probably disconnected even if it seems otherwise. */
    2222     ri->pc_last_activity = mstime();
    2223
    2224     /* Sanity check in the reply we expect, so that the code that follows
    2225      * can avoid to check for details. */
    2226     if (r->type != REDIS_REPLY_ARRAY ||
    2227         r->elements != 3 ||
    2228         r->element[0]->type != REDIS_REPLY_STRING ||
    2229         r->element[1]->type != REDIS_REPLY_STRING ||
    2230         r->element[2]->type != REDIS_REPLY_STRING ||
    2231         strcmp(r->element[0]->str,"message") != 0) return;
    2232
    2233     /* We are not interested in meeting ourselves */
    2234     if (strstr(r->element[2]->str,server.runid) != NULL) return;
    2235
    2236     sentinelProcessHelloMessage(r->element[2]->str, r->element[2]->len);
    2237 }
    ```

    有几个逻辑，

        - sentinelReceiveHelloMessages在检查reply合法性之前，即只要有reply，则更新ri->pc_last_activity,
          ri->pc_last_activity主要是用于判断pc连接是否需要reconnect的。如果距离上次更新ri->pc_last_activity
          超过3倍SENTINEL_PUBLISH_PERIOD则需要重连。这也就是pc_last_activity的全部作用。

        - 如果该hello msg是当前sentinel发出去的，则也忽略。

        - 最后处理hello msg的函数是sentinelProcessHelloMessage，后续会详细解释。

- 那么直接发送给other sentinel instance的hello msg消息，other sentinel是怎么处理的呢?

    谈到这个问题，不得不说一下sentinelcmds的相关机制。

    ```
    /* src/sentinel.c */
    385 void sentinelCommand(redisClient *c);
    386 void sentinelInfoCommand(redisClient *c);
    387 void sentinelSetCommand(redisClient *c);
    388 void sentinelPublishCommand(redisClient *c);
    389 void sentinelRoleCommand(redisClient *c);
    390
    391 struct redisCommand sentinelcmds[] = {
    392     {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
    393     {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
    394     {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},
    395     {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
    396     {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},
    397     {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
    398     {"publish",sentinelPublishCommand,3,"",0,NULL,0,0,0,0,0},
    399     {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0},
    400     {"role",sentinelRoleCommand,1,"l",0,NULL,0,0,0,0,0},
    401     {"shutdown",shutdownCommand,-1,"",0,NULL,0,0,0,0,0}
    402 };

    410 /* Perform the Sentinel mode initialization. */
    411 void initSentinel(void) {
    412     unsigned int j;
    413
    414     /* Remove usual Redis commands from the command table, then just add
    415      * the SENTINEL command. */
    416     dictEmpty(server.commands,NULL);
    417     for (j = 0; j < sizeof(sentinelcmds)/sizeof(sentinelcmds[0]); j++) {
    418         int retval;
    419         struct redisCommand *cmd = sentinelcmds+j;
    420
    421         retval = dictAdd(server.commands, sdsnew(cmd->name), cmd);
    422         redisAssert(retval == DICT_OK);
    423     }
    ```

    可以看到initSentinel这个sentinel相对redis server特有的初始化函数，首先将
    server.commands这个dict清空，然后重新加载了一批在sentinelcmds list里定义的响应命令的list。
    也就是说sentinel instance摒弃了redis server原有的所有cmd，sentinel instance单独只响应
    sentinelcmds list中命令，这个sentinelcmds list又分三类，

    - 原封不动加载的redis server提供的已有命令,
      pingCommand,subscribeCommand,unsubscribeCommand,psubscribeCommand,punsubscribeCommand,shutdownCommand
      可以看到响应ping和shutdown命令，以及subscribe相关的订阅，批量订阅，取消订阅，取消批量订阅都是
      同redis server一样的响应逻辑。

    - sentinel特有的命令，
      sentinelCommand,这个就是prefix为sentinel的那一系列命令，如sentinel is-master-down-by-addr,sentinel masters
      等的响应逻辑。

    - 被sentinel override的命令，
      sentinelPublishCommand, sentinelInfoCommand,sentinelRoleCommand

    此处介绍一下被override的sentinelPublishCommand。

    ```
    /* src/sentinel.c */
    3027 /* Our fake PUBLISH command: it is actually useful only to receive hello messages
    3028  * from the other sentinel instances, and publishing to a channel other than
    3029  * SENTINEL_HELLO_CHANNEL is forbidden.
    3030  *
    3031  * Because we have a Sentinel PUBLISH, the code to send hello messages is the same
    3032  * for all the three kind of instances: masters, slaves, sentinels. */
    3033 void sentinelPublishCommand(redisClient *c) {
    3034     if (strcmp(c->argv[1]->ptr,SENTINEL_HELLO_CHANNEL)) {
    3035         addReplyError(c, "Only HELLO messages are accepted by Sentinel instances.");
    3036         return;
    3037     }
    3038     sentinelProcessHelloMessage(c->argv[2]->ptr,sdslen(c->argv[2]->ptr));
    3039     addReplyLongLong(c,1);
    3040 }
    ```

    有几个值得注意的地方，

        - 这个publish命令响应函数仅仅用来响应其他sentinel instance发送的hello msg，对于除SENTINEL_HELLO_CHANNEL
          这个channel之外的msg, 返回addReplyError。除此之外调用sentinelProcessHelloMessage这个真正处理msg的逻辑，
          sentinelProcessHelloMessage这个函数存在的好处是和常归的hello msg处理流程做到了代码共用。

        - 通过override的做法，做到了在send hello msg给master slave redis instance以及sentinel instance的时候共用
          一个逻辑。

所以可以看到上面就是sentinel instance send hello msg以及remote instance对其的响应的两种不同的流程。
最后关于hello msg来看一下，两种不同的流程处理hello msg时共用的sentinelProcessHelloMessage的逻辑。

```
/* src/sentinel.c */
2121 void sentinelProcessHelloMessage(char *hello, int hello_len) {
2122     /* Format is composed of 8 tokens:
2123      * 0=ip,1=port,2=runid,3=current_epoch,4=master_name,
2124      * 5=master_ip,6=master_port,7=master_config_epoch. */
2125     int numtokens, port, removed, master_port;
2126     uint64_t current_epoch, master_config_epoch;
2127     char **token = sdssplitlen(hello, hello_len, ",", 1, &numtokens);
2128     sentinelRedisInstance *si, *master;
2129
2130     if (numtokens == 8) {
2131         /* Obtain a reference to the master this hello message is about */
2132         master = sentinelGetMasterByName(token[4]);
2133         if (!master) goto cleanup; /* Unknown master, skip the message. */
2134
2135         /* First, try to see if we already have this sentinel. */
2136         port = atoi(token[1]);
2137         master_port = atoi(token[6]);
2138         si = getSentinelRedisInstanceByAddrAndRunID(
2139                         master->sentinels,token[0],port,token[2]);
2140         current_epoch = strtoull(token[3],NULL,10);
2141         master_config_epoch = strtoull(token[7],NULL,10);
2142
2143         if (!si) {
2144             /* If not, remove all the sentinels that have the same runid
2145              * OR the same ip/port, because it's either a restart or a
2146              * network topology change. */
2147             removed = removeMatchingSentinelsFromMaster(master,token[0],port,
2148                             token[2]);
2149             if (removed) {
2150                 sentinelEvent(REDIS_NOTICE,"-dup-sentinel",master,
2151                     "%@ #duplicate of %s:%d or %s",
2152                     token[0],port,token[2]);
2153             }
2154
2155             /* Add the new sentinel. */
2156             si = createSentinelRedisInstance(NULL,SRI_SENTINEL,
2157                             token[0],port,master->quorum,master);
2158             if (si) {
2159                 sentinelEvent(REDIS_NOTICE,"+sentinel",si,"%@");
2160                 /* The runid is NULL after a new instance creation and
2161                  * for Sentinels we don't have a later chance to fill it,
2162                  * so do it now. */
2163                 si->runid = sdsnew(token[2]);
2164                 sentinelFlushConfig();
2165             }
2166         }
2200         /* Update the state of the Sentinel. */
2201         if (si) si->last_hello_time = mstime();
2202     }
```

- 可以看到先行判断收到的hello msg以逗号分隔后是否为8部分。如果不是，则丢弃。

- 如果用hello msg中的master_name通过sentinelGetMasterByName去sentinel.masters管辖下
的所有master信息中查找是否存在该master_name，如果找不到，即未知的master，则会被直接忽略,
此处也就是master信息不会通过hello msg的广播机制共享给其他sentinel的原因。

- 如果能够在sentinel.masters找到该master，则先行从master sentinelRedisInstance struct的
master->sentinels中删除重复的sentinel sentinelRedisInstance(如果相应的sentinel sentinelRedisInstance存在的话).
并输出了-dup-sentinel msg。然后在重新创建sentinel sentinelRedisInstance并挂载在master下。并输出+sentinel msg。
由于自动发现的sentinel的创建sentinel sentinelRedisInstance的runid就是在此处填充的，没有其他的机会填充。

- 最后可以看到此处更新了sentinel sentinelRedisInstance的last_hello_time属性。last_hello_time属性
目前仅用于addReplySentinelRedisInstance这个被用于各种"sentinel masters"这类的info逻辑中的函数里。
记录了该sentinel sentinelRedisInstance所对应的远程sentinel instance的上一条hello msg是什么时候到达的。

关于sentinelResetMaster的部分以及epoch变更的部分，后续会详细解释。

自此，hello msg的所有相关流程介绍完成。

### **各个epoch的细节(包含vote的细节)**
---------------------------------------

介绍一个epoch相关的细节，epoch其实是几种，epoch只是一个统称，epoch的作用
关乎vote，关乎upgrade config的传播，所以算是一个比较复杂的逻辑。

分别在以下数据结构里,

```
/* src/sentinel.c */
118 typedef struct sentinelRedisInstance {
122     uint64_t config_epoch;  /* Configuration epoch. */

176     char *leader;
180     uint64_t leader_epoch; /* Epoch of the 'leader' field. */
181     uint64_t failover_epoch; /* Epoch of the currently started failover. */

196 /* Main state. */
197 struct sentinelState {
198     uint64_t current_epoch;     /* Current epoch. */
```

这四个epoch之间以及他们与vote之间有着千丝万缕的联系，分开看每一个都不完整。同时顺带着会讲leader字段以及vote的逻辑。

分阶段讲吧,

- **初始化**

    ```
    /* src/sentinel.c */
    410 /* Perform the Sentinel mode initialization. */
    411 void initSentinel(void) {
    425     /* Initialize various data structures. */
    426     sentinel.current_epoch = 0;

    896 sentinelRedisInstance *createSentinelRedisInstance(char *name, int flags, char *hostname, int port, int quorum, sentinelRedisInstance *master) {
    936     ri->config_epoch = 0;
    973     /* Failover state. */
    974     ri->leader = NULL;
    975     ri->leader_epoch = 0;
    976     ri->failover_epoch = 0;
    ```

    可以明显看出来的是，current_epoch是在global sentinel struct上的一个属性，没有什么歧义。

    > /* sentinel current-epoch is a global state valid for all the masters. */

    而对于config_epoch,failover_epoch,leader_epoch来说，暂时则比较不确定，具体是作用于什么role的sentinelRedisInstance上。

- **sentinelHandleConfiguration里的一段逻辑关于epoch可以用来预热**

    ```
    /* src/sentinel.c */
    1391     } else if (!strcasecmp(argv[0],"current-epoch") && argc == 2) {
    1392         /* current-epoch <epoch> */
    1393         unsigned long long current_epoch = strtoull(argv[1],NULL,10);
    1394         if (current_epoch > sentinel.current_epoch)
    1395             sentinel.current_epoch = current_epoch;
    1396     } else if (!strcasecmp(argv[0],"config-epoch") && argc == 3) {
    1397         /* config-epoch <name> <epoch> */
    1398         ri = sentinelGetMasterByName(argv[1]);
    1399         if (!ri) return "No such master with specified name.";
    1400         ri->config_epoch = strtoull(argv[2],NULL,10);
    1401         /* The following update of current_epoch is not really useful as
    1402          * now the current epoch is persisted on the config file, but
    1403          * we leave this check here for redundancy. */
    1404         if (ri->config_epoch > sentinel.current_epoch)
    1405             sentinel.current_epoch = ri->config_epoch;
    1406     } else if (!strcasecmp(argv[0],"leader-epoch") && argc == 3) {
    1407         /* leader-epoch <name> <epoch> */
    1408         ri = sentinelGetMasterByName(argv[1]);
    1409         if (!ri) return "No such master with specified name.";
    1410         ri->leader_epoch = strtoull(argv[2],NULL,10);
    ```

    - 关于配置/* current-epoch <epoch> */，如果大于sentinel.current_epoch,则更新sentinel.current_epoch

    - 关于配置/* config-epoch <name> <epoch> */,通过name去找master，如果找到则将该master sentinelRedisInstance
      的config_epoch置为配置数，并且如果该config_epoch大于sentinel.current_epoch，则更新sentinel.current_epoch.

    - 对于/* leader-epoch <name> <epoch> */同样也是先去找到master并且将该master sentinelRedisInstance的config_epoch
      置为配置数。

- **从addReplySentinelRedisInstance捕风捉影**

    ```
    /* src/sentinel.c */
    2410 /* Redis instance to Redis protocol representation. */
    2411 void addReplySentinelRedisInstance(redisClient *c, sentinelRedisInstance *ri) {
    2509     /* Only masters */
    2510     if (ri->flags & SRI_MASTER) {
    2511         addReplyBulkCString(c,"config-epoch");
    2512         addReplyBulkLongLong(c,ri->config_epoch);
    2513         fields++;

    2578     /* Only sentinels */
    2579     if (ri->flags & SRI_SENTINEL) {
    2584         addReplyBulkCString(c,"voted-leader");
    2585         addReplyBulkCString(c,ri->leader ? ri->leader : "?");
    2586         fields++;

    2588         addReplyBulkCString(c,"voted-leader-epoch");
    2589         addReplyBulkLongLong(c,ri->leader_epoch);
    2590         fields++;
    ```

    从这里来看，config_epoch只在master sentinelRedisInstance才会被通过info信息传达出去，
    leader以及leader_epoch只在sentinel sentinelRedisInstance上会被传达出去。

接下来，就是sentinelCheckObjectivelyDown会常态化的去从每个master sentinelRedisInstance的角度
去检查挂载在master下的所有sentinel sentinelRedisInstance的SRI_MASTER_DOWN的数量，与quorum进行判断，
并判定master sentinelRedisInstance是否应该置为SRI_O_DOWN状态。此处就是quorum用来进行大多数统计的第一处逻辑。

- **sentinelAskMasterStateToOtherSentinels常态化的ask**

    ```
    /* src/sentinel.c */
    3193 void sentinelAskMasterStateToOtherSentinels(sentinelRedisInstance *master, int flags) {
    3197     di = dictGetIterator(master->sentinels);
    3198     while((de = dictNext(di)) != NULL) {
    3199         sentinelRedisInstance *ri = dictGetVal(de);
    3200         mstime_t elapsed = mstime() - ri->last_master_down_reply_time;
    3204         /* If the master state from other sentinel is too old, we clear it. */
    3205         if (elapsed > SENTINEL_ASK_PERIOD*5) {
    3206             ri->flags &= ~SRI_MASTER_DOWN;
    3207             sdsfree(ri->leader);
    3208             ri->leader = NULL;
    3209         }
    3216         if ((master->flags & SRI_S_DOWN) == 0) continue;
    3222         /* Ask */
    3223         ll2string(port,sizeof(port),master->addr->port);
    3224         retval = redisAsyncCommand(ri->cc,
    3225                     sentinelReceiveIsMasterDownReply, NULL,
    3226                     "SENTINEL is-master-down-by-addr %s %s %llu %s",
    3227                     master->addr->ip, port,
    3228                     sentinel.current_epoch,
    3229                     (master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ?
    3230                     server.runid : "*");
    ```

    可以看到在该sentinel sentinelRedisInstance的存储的remote sentinel instance对该master的评估SRI_MASTER_DOWN信息，
    如果距离上次更新到现在超过5倍SENTINEL_ASK_PERIOD时间,则直接摒弃掉该状态。如果该master sentinelRedisInstance处于
    SRI_S_DOWN，则暂时放弃从该master挂载下的所有sentinel sentinelRedisInstance去ask这一行为。

    再来看一下，在此阶段，other sentinel instance对于is-master-down-by-addr的响应逻辑。

    ```
    /* src/sentinel.c */
    2628 void sentinelCommand(redisClient *c) {
    2657     } else if (!strcasecmp(c->argv[1]->ptr,"is-master-down-by-addr")) {
    2658         /* SENTINEL IS-MASTER-DOWN-BY-ADDR <ip> <port> <current-epoch> <runid>*/
    2666         if (c->argc != 6) goto numargserr;
    2667         if (getLongFromObjectOrReply(c,c->argv[3],&port,NULL) != REDIS_OK ||
    2668             getLongLongFromObjectOrReply(c,c->argv[4],&req_epoch,NULL)
    2669                                                               != REDIS_OK)
    2670             return;
    2671         ri = getSentinelRedisInstanceByAddrAndRunID(sentinel.masters,
    2672             c->argv[2]->ptr,port,NULL);
    2673
    2674         /* It exists? Is actually a master? Is subjectively down? It's down.
    2675          * Note: if we are in tilt mode we always reply with "0". */
    2676         if (!sentinel.tilt && ri && (ri->flags & SRI_S_DOWN) &&
    2677                                     (ri->flags & SRI_MASTER))
    2678             isdown = 1;
    2679
    2680         /* Vote for the master (or fetch the previous vote) if the request
    2681          * includes a runid, otherwise the sender is not seeking for a vote. */
    2682         if (ri && ri->flags & SRI_MASTER && strcasecmp(c->argv[5]->ptr,"*")) {
    2683             leader = sentinelVoteLeader(ri,(uint64_t)req_epoch,
    2684                                             c->argv[5]->ptr,
    2685                                             &leader_epoch);
    2686         }
    2687
    2688         /* Reply with a three-elements multi-bulk reply:
    2689          * down state, leader, vote epoch. */
    2690         addReplyMultiBulkLen(c,3);
    2691         addReply(c, isdown ? shared.cone : shared.czero);
    2692         addReplyBulkCString(c, leader ? leader : "*");
    2693         addReplyLongLong(c, (long long)leader_epoch);
    ```

    - 对于c->argv[4],<current-epoch>这个参数，被用来填充req_epoch这个变量。但是由于
    strcasecmp(c->argv[5]->ptr,"*")为0的原因，填充后的req_epoch并没有派上用场。

    - 同样由于strcasecmp(c->argv[5]->ptr,"*")为0的原因，leader_epoch并不会被填充,
    leader也会不被赋值，所以addReplyLongLong返回的leader_epoch又是无意义的初始值。

    - **另外从此处可以看到sentinel instance对于未知的master的另外一部分处理逻辑，会用is-master-down-by-addr的
    <ip> <port>去当前sentinel.masters去找，如果没找到，则isdown永远为0，即对该master是否down掉并不表达意见。**
    当然如果是已知的master，并且该master sentinelRedisInstance处于SRI_S_DOWN状态，则回复isdown为1，表达出自己
    已有的对master instance的SRI_S_DOWN状态的判断。

    再来看一下，此阶段当前sentinel收到other sentinel的reply之后的callback逻辑。

    ```
    /* src/sentinel.c */
    3148 /* Receive the SENTINEL is-master-down-by-addr reply, see the
    3149  * sentinelAskMasterStateToOtherSentinels() function for more information. */
    3150 void sentinelReceiveIsMasterDownReply(redisAsyncContext *c, void *reply, void *privdata) {
    3151     sentinelRedisInstance *ri = c->data;
    3152     redisReply *r;
    3153     REDIS_NOTUSED(privdata);
    3154
    3155     if (ri) ri->pending_commands--;
    3156     if (!reply || !ri) return;
    3157     r = reply;
    3158
    3159     /* Ignore every error or unexpected reply.
    3160      * Note that if the command returns an error for any reason we'll
    3161      * end clearing the SRI_MASTER_DOWN flag for timeout anyway. */
    3162     if (r->type == REDIS_REPLY_ARRAY && r->elements == 3 &&
    3163         r->element[0]->type == REDIS_REPLY_INTEGER &&
    3164         r->element[1]->type == REDIS_REPLY_STRING &&
    3165         r->element[2]->type == REDIS_REPLY_INTEGER)
    3166     {
    3167         ri->last_master_down_reply_time = mstime();
    3168         if (r->element[0]->integer == 1) {
    3169             ri->flags |= SRI_MASTER_DOWN;
    3170         } else {
    3171             ri->flags &= ~SRI_MASTER_DOWN;
    3172         }
    3185     }
    3186 }
    ```

    在此阶段，sentinelReceiveIsMasterDownReply的用途就仅仅只是用来收集上面提到的reply响应的isdown信息，
    并记录到该master sentinelRedisInstance下相应的sentinel sentinelRedisInstance的SRI_MASTER_DOWN中。
    并且更新了该sentinel sentinelRedisInstance的ri->last_master_down_reply_time属性。

    可以看出来此阶段通过is-master-down-by这个命令沟通的信息有限。

- 发起start failover

    如果该master sentinelRedisInstance处于SRI_O_DOWN状态，则会进入sentinelStartFailover的流程。

    ```
    /* src/sentinel.c */
    3460 void sentinelStartFailover(sentinelRedisInstance *master) {
    3461     redisAssert(master->flags & SRI_MASTER);
    3462
    3463     master->failover_state = SENTINEL_FAILOVER_STATE_WAIT_START;
    3464     master->flags |= SRI_FAILOVER_IN_PROGRESS;
    3465     master->failover_epoch = ++sentinel.current_epoch;
    3471     master->failover_start_time = mstime()+rand()%SENTINEL_MAX_DESYNC;
    ```

    - 可以看到此处涉及两个epoch, 首先将++global sentinel current_epoch++，并赋给了try-failover的
    master sentinelRedisInstance的failover_epoch,

    - 此处更新了master->failover_start_time,此处就是failover_start_time的一处更新逻辑,
    并且更新时伴随着rand()%SENTINEL_MAX_DESYNC的逻辑。

    - 此处sentinel.current_epoch的更新逻辑,是当前sentinel发起的一次主动更新逻辑。

- start failover之后,sentinelAskMasterStateToOtherSentinels的更完整用途。

    - 就是sentinelAskMasterStateToOtherSentinels由于SENTINEL_ASK_FORCED这个flag加持，就ask得更频繁一些。
    并且由于(master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ?  server.runid : "*",
    此处ask的runid参数带上了当前sentinel instance的runid而具有了意义。

    - 从other sentinel reply的角度来讲，则不免由sentinelVoteLeader这个逻辑进入了选举流程,
    进入了选举流程则，local req_epoch, leader, leader_epoch则真正派上了用场。

    ```
    /* src/sentinel.c */
    2628 void sentinelCommand(redisClient *c) {
    2657     } else if (!strcasecmp(c->argv[1]->ptr,"is-master-down-by-addr")) {
    2680         /* Vote for the master (or fetch the previous vote) if the request
    2681          * includes a runid, otherwise the sender is not seeking for a vote. */
    2682         if (ri && ri->flags & SRI_MASTER && strcasecmp(c->argv[5]->ptr,"*")) {
    2683             leader = sentinelVoteLeader(ri,(uint64_t)req_epoch,
    2684                                             c->argv[5]->ptr,
    2685                                             &leader_epoch);
    2686         }
    2687
    2688         /* Reply with a three-elements multi-bulk reply:
    2689          * down state, leader, vote epoch. */
    2690         addReplyMultiBulkLen(c,3);
    2691         addReply(c, isdown ? shared.cone : shared.czero);
    2692         addReplyBulkCString(c, leader ? leader : "*");
    2693         addReplyLongLong(c, (long long)leader_epoch);
    ```

    此处为other sentinel响应is-master-down-by-addr命令时的vote逻辑，other sentinel执行sentinelVoteLeader采用了
    作为参数的当前sentinel的current_epoch(也即为当前sentinel发起的这轮failover的failover_epoch),
    来评估自己的投票选择，评估二字为何，Vote for the master (or fetch the previous vote)就是解释。
    可以看到此阶段此处other sentinel进入sentinelVoteLeader并不要求other sentinel对该
    master sentinelRedisInstance有除了role之外的任何要求。

    此处会岔开去讲sentinelVoteLeader这个重头戏，等会回过来头讲此阶段下当前sentinel的reply callback的逻辑。

- other sentinel sentinelVoteLeader reply给当前sentinel

    接下来这一段内容中,我临时设身处地成other sentinel的角度，接下来的这段内容会临时将other sentinel称为当前sentinel。
    将other sentinel切换到第一人称视角。为了表达方便。

    ```
    /* src/sentinel.c */
    3243 char *sentinelVoteLeader(sentinelRedisInstance *master, uint64_t req_epoch, char *req_runid, uint64_t *leader_epoch) {
    3244     if (req_epoch > sentinel.current_epoch) {
    3249         sentinel.current_epoch = req_epoch;
    3253     }
    3254
    3255     if (master->leader_epoch < req_epoch && sentinel.current_epoch <= req_epoch)
    3256     {
    3257         mstime_t time_since_last_vote = mstime() - master->failover_start_time;
    3266         if (time_since_last_vote > master->failover_timeout ||
    3267             strcasecmp(req_runid,server.runid) == 0 ||
    3268             master->leader == NULL) {
    3269             sdsfree(master->leader);
    3270             master->leader = sdsnew(req_runid);
    3271         }
    3272         master->leader_epoch = sentinel.current_epoch;
    3276         /* If we did not voted for ourselves, set the master failover start
    3277          * time to now, in order to force a delay before we can start a
    3278          * failover for the same master. */
    3279         if (strcasecmp(master->leader,server.runid)) {
    3280             mstime_t last_time = master->failover_start_time;
    3281             master->failover_start_time = mstime()+rand()%SENTINEL_MAX_DESYNC;
    3286         }
    3287     }
    3288
    3289     *leader_epoch = master->leader_epoch;
    3290     return master->leader ? sdsnew(master->leader) : NULL;
    3291 }
    ```

    - 如果该req_epoch大于当前的sentinel.current_epoch,则更新当前sentinel的sentinel.current_epoch。
    此处为被动更新sentinel.current_epoch的一个逻辑。

    - 如果master sentinelRedisInstance的leader_epoch小于该req_epoch并且当前sentinel的sentinel.current_epoch
    不大于req_epoch,其实可以看到由于上面的逻辑，根本没有小于的可能。

        - 上面两个条件满足则考虑更新该master sentinelRedisInstance的leader信息，
          将该req_runid参数赋给master sentinelRedisInstance的leader属性。
          例外就是如果距离该master sentinelRedisInstance
          的failover_start_time还没有超过failover_timeout这么长时间，
          此处就是failover_start_time的一个限制逻辑。
          则坚持上次的投票意见不变,例外就是投票给自己,投票给自己其实不是sentinelVoteLLeader在这个阶段的作用。
          投票给自己后续会解释。

        - 更新master->leader_epoch为当前的sentinel.current_epoch

        - 并且如果我们并不是投票给了自己，则还有一个约束条件就是去更新master->failover_start_time,
          限制自己下次vote或者start failover的时间。此处就是failover_start_time的一个更新逻辑，当然会影响限制逻辑。
          可以看到更新failover_start_time伴随着一个rand()%SENTINEL_MAX_DESYNC的逻辑。这是一个比较无力的failover_start_time的desync逻辑。
          至此，failover_start_time的两处更新逻辑以及一处限制逻辑都已经讲到了，并且两处
          更新failover_start_time都伴随着一个rand()%SENTINEL_MAX_DESYNC的逻辑.

        - 刚好提一下failover_start_time的最后一处限制逻辑。

            ```
            /* src/sentinel.c */
            3491 int sentinelStartFailoverIfNeeded(sentinelRedisInstance *master) {
            3500     /* Last failover attempt started too little time ago? */
            3501     if (now - master->failover_start_time <
            3502         master->failover_timeout*2)
            3503     {
            3504         if (master->failover_delay_logged != master->failover_start_time) {
            3505             time_t clock = (master->failover_start_time +
            3506                             master->failover_timeout*2) / 1000;
            3507             char ctimebuf[26];
            3508
            3509             ctime_r(&clock,ctimebuf);
            3510             ctimebuf[24] = '\0'; /* Remove newline. */
            3511             master->failover_delay_logged = master->failover_start_time;
            3512             redisLog(REDIS_WARNING,
            3513                 "Next failover delay: I will not start a failover before %s",
            3514                 ctimebuf);
            3515         }
            3516         return 0;
            ```

            此处是start failover if needed的一个前置条件，如果距离上次更新该master sentinelRedisInstance的failover_start_time
            还没超过2倍failover_timeout,则直接return，则暂时不进入start failover.

    - 最后通过return值以及填充leader_epoch参数的这俩个方式，将此次投票信息返回出去。

    整个过程就是将同意发起start failover的sentinel instance在is-master-down-by-addr在参数中带上的current_epoch信息，即req_epoch参数。
    为什么同意呢，因为当前sentinel中该master sentinelRedisInstance的leader_epoch小于该值，并且当前sentinel的current_epoch还没有
    超前于该req_epoch,关于current_epoch更新的其他细节后续还会解释，
    此处的current_epoch检查逻辑蔑视了比它小的req_epoch要求更新投票信息的请求，
    则此时更新投票信息，包括该master sentinelRedisInstance的leader以及leader_epoch属性。
    至此临时的other sentinel的第一视角结束。

- vote reply callback sentinelReceiveIsMasterDownReply的更完整的作用

Reply with a three-elements multi-bulk reply: down state, leader, vote epoch

other sentinel vote reply信息中带上了leader以及vote epoch信息。

```
/* src/sentinel.c */
3150 void sentinelReceiveIsMasterDownReply(redisAsyncContext *c, void *reply, void *privdata) {
3151     sentinelRedisInstance *ri = c->data;
3158
3159     /* Ignore every error or unexpected reply.
3160      * Note that if the command returns an error for any reason we'll
3161      * end clearing the SRI_MASTER_DOWN flag for timeout anyway. */
3162     if (r->type == REDIS_REPLY_ARRAY && r->elements == 3 &&
3163         r->element[0]->type == REDIS_REPLY_INTEGER &&
3164         r->element[1]->type == REDIS_REPLY_STRING &&
3165         r->element[2]->type == REDIS_REPLY_INTEGER)
3166     {
3173         if (strcmp(r->element[1]->str,"*")) {
3174             /* If the runid in the reply is not "*" the Sentinel actually
3175              * replied with a vote. */
3176             sdsfree(ri->leader);
3182             ri->leader = sdsnew(r->element[1]->str);
3183             ri->leader_epoch = r->element[2]->integer;
3184         }
3185     }
3186 }
```

此处当前sentinel对other sentinel的投票信息的处理是将leader,leader_epoch信息,
直接存入sentinel sentinelRedisInstance的leader和leader_epoch中。
当然此处的sentinel sentinelRedisInstance是挂载在当前正在进行failover的master sentinelRedisInstance下。
可以看到在当前sentinel以及other sentinel中对于leader和leader_epoch是存储在不同role的sentinelRedisInstance中的。

- start failover之后，正在开始failover的流程之前，叫做wait start failover

```
/* src/sentinel.c */
3633 void sentinelFailoverWaitStart(sentinelRedisInstance *ri) {
3634     char *leader;
3635     int isleader;
3636
3637     /* Check if we are the leader for the failover epoch. */
3638     leader = sentinelGetLeader(ri, ri->failover_epoch);
3639     isleader = leader && strcasecmp(leader,server.runid) == 0;
3640     sdsfree(leader);
3641
3644     if (!isleader && !(ri->flags & SRI_FORCE_FAILOVER)) {
3645         int election_timeout = SENTINEL_ELECTION_TIMEOUT;
3646
3649         if (election_timeout > ri->failover_timeout)
3650             election_timeout = ri->failover_timeout;
3651         /* Abort the failover if I'm not the leader after some time. */
3652         if (mstime() - ri->failover_start_time > election_timeout) {
3656             sentinelAbortFailover(ri);
3657         }
3658         return;
```

可以看到此处会从当前sentinel来统计vote情况，上一步的vote reply已经存储在
当前master sentinelRedisInstance挂载下的sentinel sentinelRedisInstance中了，
如果选举失败，则此阶段最终会进入election timeout状态.

详细介绍一下sentinelGetLeader，

先看前半部分，统计已有的vote,看是否有winner

```
3316 /* Scan all the Sentinels attached to this master to check if there                                                                                                                                   3317  * is a leader for the specified epoch.                                                                                                                                                               3318  *
3319  * To be a leader for a given epoch, we should have the majority of
3320  * the Sentinels we know (ever seen since the last SENTINEL RESET) that
3321  * reported the same instance as leader for the same epoch. */
/* src/sentinel.c */
3322 char *sentinelGetLeader(sentinelRedisInstance *master, uint64_t epoch) {
3333     counters = dictCreate(&leaderVotesDictType,NULL);
3335     voters = dictSize(master->sentinels)+1; /* All the other sentinels and me. */
3336
3337     /* Count other sentinels votes */
3338     di = dictGetIterator(master->sentinels);
3339     while((de = dictNext(di)) != NULL) {
3340         sentinelRedisInstance *ri = dictGetVal(de);
3341         if (ri->leader != NULL && ri->leader_epoch == epoch) {
3342             sentinelLeaderIncr(counters,ri->leader);
3348         }
3349     }
3350     dictReleaseIterator(di);
3351
3355     di = dictGetIterator(counters);
3356     while((de = dictNext(di)) != NULL) {
3357         uint64_t votes = dictGetUnsignedIntegerVal(de);
3358
3359         if (votes > max_votes) {
3360             max_votes = votes;
3361             winner = dictGetKey(de);
3362         }
3363     }
3364     dictReleaseIterator(di);
```

可以看到此处就是all the Sentinels attached to this master
统计这些sentinel sentinelRedisInstance的leader leader_epoch信息和参数given epoch是否吻合,
并sentinelLeaderIncr累加到counters dict中。
**这个given epoch其实就是master  sentinelRedisInstance的failover_epoch了，不一定是sentinel.current_epoch，
可能此时当前sentinel的current_epoch已经由于接下来的failover又++了**

接着开始统计counters这个dict,将投票最多的runid记录到winner中，将该投票记录到max_votes中.

再看后半部分，统计已有的vote,看是否有winner

```
/* src/sentinel.c */
3322 char *sentinelGetLeader(sentinelRedisInstance *master, uint64_t epoch) {
3366     /* Count this Sentinel vote:
3367      * if this Sentinel did not voted yet, either vote for the most
3368      * common voted sentinel, or for itself if no vote exists at all. */
3369     if (winner)
3370         myvote = sentinelVoteLeader(master,epoch,winner,&leader_epoch);
3371     else
3372         myvote = sentinelVoteLeader(master,epoch,server.runid,&leader_epoch);
3373
3374     if (myvote && leader_epoch == epoch) {
3375         uint64_t votes = sentinelLeaderIncr(counters,myvote);
3376
3377         if (votes > max_votes) {
3378             max_votes = votes;
3379             winner = myvote;
3380         }
3381     }
3382
3383     voters_quorum = voters/2+1;
3384     if (winner && (max_votes < voters_quorum || max_votes < master->quorum))
3385         winner = NULL;
3386
3387     winner = winner ? sdsnew(winner) : NULL;
3388     sdsfree(myvote);
3389     dictRelease(counters);
3390     return winner;
```

如果在统计中，有一个winner出现，则当前sentinel通过sentinelVoteLeader投给自己。
当然如果当前sentinel已经sentinelVoteLeader过了，则此处不会算入,重复投票。
此处并非之前给员外讲过的羊群效应，因为vote信息并没有广而告知，并没有广播传播。
仅仅自己当前sentinel的一点私心而已,当然其实也是很慷慨的，如果有人已经赢得了时间获得了投票，
至少他这一票肯定会投给他。

最终统计vote情况，需要大于大多数，也需要大于master->quorum。此处也就是quorum的另外一处大多数统计的用途。
如果不满足，则即使有winner也会被清空。
此处sentinelGetLeader一次统计不成功，会再次统计，一直重试，直到SENTINEL_ELECTION_TIMEOUT,大概10s.
关于leader以及leader_epoch上面大致已经介绍完了。

- failover中SENTINEL_FAILOVER_STATE_WAIT_PROMOTION状态

```
/* src/sentinel.c */
1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
1945     if ((ri->flags & SRI_SLAVE) && role == SRI_MASTER) {
1946         /* If this is a promoted slave we can change state to the
1947          * failover state machine. */
1948         if ((ri->flags & SRI_PROMOTED) &&
1949             (ri->master->flags & SRI_FAILOVER_IN_PROGRESS) &&
1950             (ri->master->failover_state ==
1951                 SENTINEL_FAILOVER_STATE_WAIT_PROMOTION))
1952         {
1958             ri->master->config_epoch = ri->master->failover_epoch;
1959             ri->master->failover_state = SENTINEL_FAILOVER_STATE_RECONF_SLAVES;
```

**可以看到当前sentinel的master sentinelRedisInstance的config_epoch被更新为master sentinelRedisInstance的
failover_epoch,虽然是在借助slave sentinelRedisInstance的情况下完成的,
但是此举也就是认定config upgrade的这一重要步骤。**
此处master config_epoch upgrade之后，新的master config_epoch以及promoted slave的ip和port信息,以及当前sentinel.current_epoch就会不断
通过send hello msg从当前sentinel广播出去，虽然当前sentinel都还未真正生效此变更，因为还不到当前sentinel变更的时候，
可以解释为，此举之后当前sentinel的行为是不可逆的，一定要成功，即使真的crash了，那么这个upgrade config也由于
广播出去了，会被其他sentinel最终fix生效。但是当前sentinel还需要做在之前的视角上做一些事情，所以还不到变更的时机。

- other sentinel收到hello msg的处理逻辑sentinelProcessHelloMessage

```
/* src/sentinel.c */
2121 void sentinelProcessHelloMessage(char *hello, int hello_len) {
2122     /* Format is composed of 8 tokens:
2123      * 0=ip,1=port,2=runid,3=current_epoch,4=master_name,
2124      * 5=master_ip,6=master_port,7=master_config_epoch. */
2126     uint64_t current_epoch, master_config_epoch;
2129
2130     if (numtokens == 8) {
2132         master = sentinelGetMasterByName(token[4]);
2133         if (!master) goto cleanup; /* Unknown master, skip the message. */
2134
2135         /* First, try to see if we already have this sentinel. */
2137         master_port = atoi(token[6]);
2138         si = getSentinelRedisInstanceByAddrAndRunID(
2139                         master->sentinels,token[0],port,token[2]);
2140         current_epoch = strtoull(token[3],NULL,10);
2141         master_config_epoch = strtoull(token[7],NULL,10);
2168         /* Update local current_epoch if received current_epoch is greater.*/
2169         if (current_epoch > sentinel.current_epoch) {
2170             sentinel.current_epoch = current_epoch;
2174         }
2175
2176         /* Update master info if received configuration is newer. */
2177         if (master->config_epoch < master_config_epoch) {
2178             master->config_epoch = master_config_epoch;
2179             if (master_port != master->addr->port ||
2180                 strcmp(master->addr->ip, token[5]))
2181             {
2182                 sentinelAddr *old_addr;
2183
2191                 old_addr = dupSentinelAddr(master->addr);
2192                 sentinelResetMasterAndChangeAddress(master, token[5], master_port);
```

可以看到此处有几个更新逻辑。

    - 如果hello msg 的current_epoch,大于sentinel.current_epoch，则更新sentinel.current_epoch，
      这里是sentinel.current_epoch更新的有一处逻辑。此处也就是current_epoch的最后一处更新逻辑.

    - 如果hello msg的master_config_epoch大于master->config_epoch，则此处更新master->config_epoch

    - 在master config_epoch变更的情况下，如果master ip port和当前不匹配，则做sentinelResetMasterAndChangeAddress切换。

- other sentinel在获取该hello msg之，以及当前sentinel在准备好switch状态后，sentinelResetMaster
    
    - sentinelResetMaster清空了master sentinelRedisInstance的leader信息。

    - 将failover_start_time置为0，去掉了之前提到的failover_start_time的影响。

    - 将failover_state置为初始SENTINEL_FAILOVER_STATE_NONE值。

    - 清空了promoted_slave信息。
    
    - **关于epoch信息是完全保留的**,除该master sentinelRedisInstance的leader_epoch由于leader被清空以失效之外。

    - sentinelResetMasterAndChangeAddress在sentinelResetMaster之后switch了master->addr
