## **sentinel failover过程的详细介绍**
--------------------------------------

### **sentinel主逻辑函数**
-------------------------------------

sentinel逻辑的入口，sentinel主观能动所进行的所有动作由此触发。

```
/* src/sentinel.c */
4010 void sentinelTimer(void) {
4011     sentinelCheckTiltCondition();
4012     sentinelHandleDictOfRedisInstances(sentinel.masters);

3953 /* Perform scheduled operations for all the instances in the dictionary.
3954  * Recursively call the function against dictionaries of slaves. */
3955 void sentinelHandleDictOfRedisInstances(dict *instances) {
3956     dictIterator *di;
3957     dictEntry *de;
3958     sentinelRedisInstance *switch_to_promoted = NULL;
3959
3960     /* There are a number of things we need to perform against every master. */
3961     di = dictGetIterator(instances);
3962     while((de = dictNext(di)) != NULL) {
3963         sentinelRedisInstance *ri = dictGetVal(de);
3964
3965         sentinelHandleRedisInstance(ri);
3966         if (ri->flags & SRI_MASTER) {
3967             sentinelHandleDictOfRedisInstances(ri->slaves);
3968             sentinelHandleDictOfRedisInstances(ri->sentinels);

3918 /* Perform scheduled operations for the specified Redis instance. */
3919 void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {
3920     /* ========== MONITORING HALF ============ */
3921     /* Every kind of instance */
3922     sentinelReconnectInstance(ri);
3923     sentinelSendPeriodicCommands(ri);
3924
3925     /* ============== ACTING HALF ============= */
3934
3935     /* Every kind of instance */
3936     sentinelCheckSubjectivelyDown(ri);
3937
3938     /* Masters and slaves */
3939     if (ri->flags & (SRI_MASTER|SRI_SLAVE)) {
3940         /* Nothing so far. */
3941     }
3942
3943     /* Only masters */
3944     if (ri->flags & SRI_MASTER) {
3945         sentinelCheckObjectivelyDown(ri);
3946         if (sentinelStartFailoverIfNeeded(ri))
3947             sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);
3948         sentinelFailoverStateMachine(ri);
3949         sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);
3950     }
3951 }
```

sentinelHandleRedisInstance里的函数很有意思，后续章节会逐一介绍。

### **sentinel monitor常态信息的收集**
-------------------------------------

这一章节介绍sentinelHandleRedisInstance里的两个函数

- sentinelReconnectInstance

	首先介绍一下由sentinel所产生的tcp连接，以及tcp连接重连的机制。

	相关的数据结构，

	```
	/* src/sentinel.c */
	57 #define SRI_DISCONNECTED (1<<3)
	
	118 typedef struct sentinelRedisInstance {
	124     redisAsyncContext *cc; /* Hiredis context for commands. */
	125     redisAsyncContext *pc; /* Hiredis context for Pub / Sub. */
	127     mstime_t cc_conn_time; /* cc connection time. */
	128     mstime_t pc_conn_time; /* pc connection time. */
	```
	cc是连接到的sentinelRedisInstance具体所指向的instance的command connection，pc是用于subscribe sentinelRedisInstance具体所指向的instance的pubsub connection。
	
	从之前介绍的sentinel的拓扑结构结合此处来看，每个master sentinelRedisInstance struct会创建一个cc连接和一个pc连接到指向的相应的redis instance，每个master sentinelRedisInstance struct的\*slaves dict所指向的所有slave sentinelRedisInstance struct也会一一创建一个cc连接和一个pc连接到指向的相应的redis instance。另外对于每个master sentinelRedisInstance struct的\*sentinels dict所指向的所有sentinel sentinelRedisInstance也会一一创建一个cc连接到指向的sentinel instance（没有pc，sentinel不会直接订阅其他的sentinels）。**值得注意的是，拓扑结构中的sentinel部分，对于每个master，监控该master的所有sentinel, 都会直接创建一个role为sentinel的sentinelRedisInstance并挂载在该master下，而不是不同master之间共享同一个sentinel sentinelRedisInstance，这将会导致一个很严重的连接数增长的问题，后续会详细介绍**。

	初始化连接以及连接状态，可以看出来一开始都置为disconnected状态

	```
	/* src/sentinel.c */
	931     /* Note that all the instances are started in the disconnected state,
	932      * the event loop will take care of connecting them. */
	933     ri->flags = flags | SRI_DISCONNECTED;
	938     ri->cc = NULL;
	939     ri->pc = NULL;
	940     ri->pending_commands = 0;
	941     ri->cc_conn_time = 0;
	942     ri->pc_conn_time = 0;
	```

	干掉无效连接
	
	```
	/* src/sentinel.c */
	1619 /* Completely disconnect a hiredis link from an instance. */
	1620 void sentinelKillLink(sentinelRedisInstance *ri, redisAsyncContext *c) {
	1621     if (ri->cc == c) {
	1622         ri->cc = NULL;
	1623         ri->pending_commands = 0;
	1624     }
	1625     if (ri->pc == c) ri->pc = NULL;
	1626     c->data = NULL;
	1627     ri->flags |= SRI_DISCONNECTED;
	1628     redisAsyncFree(c);
	1629 }
	```
	
	cc重连机制
	
	```
	/* src/sentinel.c */
	1703 /* Create the async connections for the specified instance if the instance
	1704  * is disconnected. Note that the SRI_DISCONNECTED flag is set even if just
	1705  * one of the two links (commands and pub/sub) is missing. */
	1706 void sentinelReconnectInstance(sentinelRedisInstance *ri) {
	1707     if (!(ri->flags & SRI_DISCONNECTED)) return;
	1708
	1709     /* Commands connection. */
	1710     if (ri->cc == NULL) {
	1711         ri->cc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,REDIS_BIND_ADDR);
	1712         if (ri->cc->err) {
	1713             sentinelEvent(REDIS_DEBUG,"-cmd-link-reconnection",ri,"%@ #%s",
	1714                 ri->cc->errstr);
	1715             sentinelKillLink(ri,ri->cc);
	1716         } else {
	1717             redisLog(REDIS_VERBOSE,
	1718                 "+cmd-link-connection %s %d",
	1719                 ri->addr->ip, ri->addr->port);
	1720             ri->cc_conn_time = mstime();
	1721             ri->cc->data = ri;
	1722             redisAeAttach(server.el,ri->cc);
	1723             redisAsyncSetConnectCallback(ri->cc,
	1724                                             sentinelLinkEstablishedCallback);
	1725             redisAsyncSetDisconnectCallback(ri->cc,
	1726                                             sentinelDisconnectCallback);
	1727             sentinelSendAuthIfNeeded(ri,ri->cc);
	1728             sentinelSetClientName(ri,ri->cc,"cmd");
	```
	可以看到是通过SRI_DISCONNECTED 这个flag来判断是否要进一步判断重连，这个flag的意思就是cc或者pc只要出问题了，都为true，都需要重连。然后是判断ri->cc是否为NULL。由于是异步bind，还设置了sentinelLinkEstablishedCallback，sentinelDisconnectCallback两个callback。还有一个值得注意的地方，sentinelSetClientName会让远程instance 将这个connection rename更有意义的名字，供远程instance更好的管理连接。
	
	```
	/* src/sentinel.c */
	1686 /* Use CLIENT SETNAME to name the connection in the Redis instance as
	1687  * sentinel-<first_8_chars_of_runid>-<connection_type>
	1688  * The connection type is "cmd" or "pubsub" as specified by 'type'.
	1689  *
	1690  * This makes it possible to list all the sentinel instances connected
	1691  * to a Redis servewr with CLIENT LIST, grepping for a specific name format. */
	1692 void sentinelSetClientName(sentinelRedisInstance *ri, redisAsyncContext *c, char *type) {
	1693     char name[64];
	1694
	1695     snprintf(name,sizeof(name),"sentinel-%.8s-%s",server.runid,type);
	1696     if (redisAsyncCommand(c, sentinelDiscardReplyCallback, NULL,
	1697         "CLIENT SETNAME %s", name) == REDIS_OK)
	1698     {
	1699         ri->pending_commands++;
	1700     }
	1701 }
	```
	
	pc重连机制
	
	```
	/* src/sentinel.c */
	1706 void sentinelReconnectInstance(sentinelRedisInstance *ri) {
	1707     if (!(ri->flags & SRI_DISCONNECTED)) return;
	1734     /* Pub / Sub */
	1735     if ((ri->flags & (SRI_MASTER|SRI_SLAVE)) && ri->pc == NULL) {
	1736         ri->pc = redisAsyncConnectBind(ri->addr->ip,ri->addr->port,REDIS_BIND_ADDR);
	1737         if (ri->pc->err) {
	1738             sentinelEvent(REDIS_DEBUG,"-pubsub-link-reconnection",ri,"%@ #%s",
	1739                 ri->pc->errstr);
	1740             sentinelKillLink(ri,ri->pc);
	1741         } else {
	1742             int retval;
	1743
	1744             redisLog(REDIS_VERBOSE,
	1745                 "+pubsub-link-connection %s %d",
	1746                 ri->addr->ip, ri->addr->port);
	1747             ri->pc_conn_time = mstime();
	1748             ri->pc->data = ri;
	1749             redisAeAttach(server.el,ri->pc);
	1750             redisAsyncSetConnectCallback(ri->pc,
	1751                                             sentinelLinkEstablishedCallback);
	1752             redisAsyncSetDisconnectCallback(ri->pc,
	1753                                             sentinelDisconnectCallback);
	1754             sentinelSendAuthIfNeeded(ri,ri->pc);
	1755             sentinelSetClientName(ri,ri->pc,"pubsub");
	1756             /* Now we subscribe to the Sentinels "Hello" channel. */
	1757             retval = redisAsyncCommand(ri->pc,
	1758                 sentinelReceiveHelloMessages, NULL, "SUBSCRIBE %s",
	1759                     SENTINEL_HELLO_CHANNEL);
	1760             if (retval != REDIS_OK) {
	1761                 /* If we can't subscribe, the Pub/Sub connection is useless
	1762                  * and we can simply disconnect it and try again. */
	1763                 sentinelKillLink(ri,ri->pc);
	1764                 return;
	1765             }
	1766         }
	1767     }
	1768     /* Clear the DISCONNECTED flags only if we have both the connections
	1769      * (or just the commands connection if this is a sentinel instance). */
	1770     if (ri->cc && (ri->flags & SRI_SENTINEL || ri->pc))
	1771         ri->flags &= ~SRI_DISCONNECTED;
	```
	可以看到是判断了 ((ri->flags & (SRI_MASTER|SRI_SLAVE)) && ri->pc == NULL)，并且设置了callback，并且设置了connection name，并且立即通过pc连接订阅了instance上的SENTINEL_HELLO_CHANNEL这个频道，关于hello message后续章节会详细介绍，**所以sentinel只是同master或者slave instance建立了pc连接，并且只订阅了SENTINEL_HELLO_CHANNEL这个频道，用于被动的接受新鲜的消息。**
	
	有几个flag需要注意：
	
	- SRI_MASTER 表示这个sentinelRedisInstance struct表示的role是master
	
	- SRI_SLAVE 表示这个sentinelRedisInstance struct表示的role是slave
	
	- SRI_SENTINEL 表示这个sentinelRedisInstance struct表示的role是sentinel
	
	- SRI_DISCONNECTED 对于表示master或者slave的sentinelRedisInstance struct（即ri->flags & (SRI_MASTER|SRI_SLAVE)来说，是指cc或者pc任意一个处于未响应的状态，对于ri->flags & SRI_SENTINEL来讲是指cc处于未响应的状态。
	
- sentinelSendPeriodicCommands
	相关常量定义如下：
	
	```
	/* src/sentinel.c */
	71 /* Note: times are in milliseconds. */
  	72 #define SENTINEL_INFO_PERIOD 10000
  	73 #define SENTINEL_PING_PERIOD 1000
  	74 #define SENTINEL_ASK_PERIOD 1000
  	75 #define SENTINEL_PUBLISH_PERIOD 2000
	```
	
	```
	/* src/sentinel.c */
	2342 /* Send periodic PING, INFO, and PUBLISH to the Hello channel to
	2343  * the specified master or slave instance. */
	2344 void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {
	2345     mstime_t now = mstime();
	2346     mstime_t info_period, ping_period;
	2347     int retval;
	2348
	2349     /* Return ASAP if we have already a PING or INFO already pending, or
	2350      * in the case the instance is not properly connected. */
	2351     if (ri->flags & SRI_DISCONNECTED) return;
	2352
	2353     /* For INFO, PING, PUBLISH that are not critical commands to send we
	2354      * also have a limit of SENTINEL_MAX_PENDING_COMMANDS. We don't
	2355      * want to use a lot of memory just because a link is not working
	2356      * properly (note that anyway there is a redundant protection about this,
	2357      * that is, the link will be disconnected and reconnected if a long
	2358      * timeout condition is detected. */
	2359     if (ri->pending_commands >= SENTINEL_MAX_PENDING_COMMANDS) return;
	2360
	2361     /* If this is a slave of a master in O_DOWN condition we start sending
	2362      * it INFO every second, instead of the usual SENTINEL_INFO_PERIOD
	2363      * period. In this state we want to closely monitor slaves in case they
	2364      * are turned into masters by another Sentinel, or by the sysadmin. */
	2365     if ((ri->flags & SRI_SLAVE) &&
	2366         (ri->master->flags & (SRI_O_DOWN|SRI_FAILOVER_IN_PROGRESS))) {
	2367         info_period = 1000;
	2368     } else {
	2369         info_period = SENTINEL_INFO_PERIOD;
	2370     }
	2371
	2372     /* We ping instances every time the last received pong is older than
	2373      * the configured 'down-after-milliseconds' time, but every second
	2374      * anyway if 'down-after-milliseconds' is greater than 1 second. */
	2375     ping_period = ri->down_after_period;
	2376     if (ping_period > SENTINEL_PING_PERIOD) ping_period = SENTINEL_PING_PERIOD;
	```
	先看sentinelSendPeriodicCommands的前半部分，如果是SRI_DISCONNECTED，就跳过。
	
	并且设置了一个最大的pending commands的上限, 默认是100，达到上限之后，不再增加，除非老的未响应的连接超时被杀掉腾出位置来。从这里可以看出，如果监控一批100个以上的redis instances，这个上限还是很紧张的。
	
	info_period默认是10000ms，如果这个sentinelRedisInstance是表示slave的，并且该slave的master处于odown甚至是failover时，info_period是1000ms，以更快得得到该slave可能被提升为master而产生的角色变更信息。
	
	ping_period是指min(1000ms,down-after-milliseconds)
	
	```
	/* src/sentinel.c */
	2342 /* Send periodic PING, INFO, and PUBLISH to the Hello channel to
	2343  * the specified master or slave instance. */
	2344 void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {
	2345     mstime_t now = mstime();
	2378     if ((ri->flags & SRI_SENTINEL) == 0 &&
	2379         (ri->info_refresh == 0 ||                                                                                                                                                                    	2380         (now - ri->info_refresh) > info_period))
	2381     {
	2382         /* Send INFO to masters and slaves, not sentinels. */
	2383         retval = redisAsyncCommand(ri->cc,
	2384             sentinelInfoReplyCallback, NULL, "INFO");
	2385         if (retval == REDIS_OK) ri->pending_commands++;
	2386     } else if ((now - ri->last_pong_time) > ping_period) {
	2387         /* Send PING to all the three kinds of instances. */
	2388         sentinelSendPing(ri);
	2389     } else if ((now - ri->last_pub_time) > SENTINEL_PUBLISH_PERIOD) {
	2390         /* PUBLISH hello messages to all the three kinds of instances. */
	2391         sentinelSendHello(ri);
	2392     }
	```

	可以看到info只是作用于master或者slave角色的sentinelRedisInstance struct上，sentinelSendHello后续章节会详细解释，这里解释一下sentinelInfoReplyCallback, sentinelSendPing这两个func。
	
	```
	/* src/sentinel.c */
	2038 void sentinelInfoReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {
	2047     if (r->type == REDIS_REPLY_STRING) {
	2048         sentinelRefreshInstanceInfo(ri,r->str);
	2049     }
	2050 }	
	```
	由于这是一个callback方法，ri其实就是上面redisAsyncCommand的第一个参数相关的sentinelRedisInstance，只能表示master或者slave。
	
	```
	/* src/sentinel.c */
	1789 /* Process the INFO output from masters. */
	1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
	1795     /* cache full INFO output for instance */
	1796     sdsfree(ri->info);
	1797     ri->info = sdsnew(info);
	1911     ri->info_refresh = mstime();
	```
	可以看到ri->info的作用就是用来直接缓存整个从指向的redis instance获取到的info的。ri->info_refresh只是记录了更新info的之后的那个时间点。
	
	```
	/* src/sentinel.c */
	1789 /* Process the INFO output from masters. */
	1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
	1822         /* old versions: slave0:<ip>,<port>,<state>
	1823          * new versions: slave0:ip=127.0.0.1,port=9999,... */
	1824         if ((ri->flags & SRI_MASTER) &&
	1825             sdslen(l) >= 7 &&
	1826             !memcmp(l,"slave",5) && isdigit(l[5]))
	1827         {
	1850             /* Check if we already have this slave into our table,
	1851              * otherwise add it. */
	1852             if (sentinelRedisInstanceLookupSlave(ri,ip,atoi(port)) == NULL) {
	1853                 if ((slave = createSentinelRedisInstance(NULL,SRI_SLAVE,ip,
	1854                             atoi(port), ri->quorum, ri)) != NULL)
	```
	之前也提到过，从master的info中发现该master下未知的slave，若有发现，则作为slave挂载在当前master sentinelRedisInstance之下。
	
	```
	/* src/sentinel.c */
	1789 /* Process the INFO output from masters. */
	1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
	1861         /* master_link_down_since_seconds:<seconds> */
	1862         if (sdslen(l) >= 32 &&
	1863             !memcmp(l,"master_link_down_since_seconds",30))
	1864         {
	1865             ri->master_link_down_time = strtoll(l+31,NULL,10)*1000;
	1866         }
	```
	master_link_down_since_seconds:<seconds> 这是slave instance的info中的一部分，master instance的info中没有部分信息。也就只有表示slave的sentinelRedisInstance struct的这个信息才是有效的。
    slave的sentinelRedisInstance struct 会将该信息记录在ri->master_link_down_time中。关于master_link_down_since_seconds的细节后续会详细介绍。
	
	```
	/* src/sentinel.c */
	1789 /* Process the INFO output from masters. */
	1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
	1868         /* role:<role> */
	1869         if (!memcmp(l,"role:master",11)) role = SRI_MASTER;
	1870         else if (!memcmp(l,"role:slave",10)) role = SRI_SLAVE;
	1871
	1872         if (role == SRI_SLAVE) {
	1873             /* master_host:<host> */
	1874             if (sdslen(l) >= 12 && !memcmp(l,"master_host:",12)) {
	1875                 if (ri->slave_master_host == NULL ||
	1876                     strcasecmp(l+12,ri->slave_master_host))
	1877                 {
	1878                     sdsfree(ri->slave_master_host);
	1879                     ri->slave_master_host = sdsnew(l+12);
	1880                     ri->slave_conf_change_time = mstime();
	1881                 }
	1882             }
	1883
	1884             /* master_port:<port> */
	1885             if (sdslen(l) >= 12 && !memcmp(l,"master_port:",12)) {
	1886                 int slave_master_port = atoi(l+12);
	1887
	1888                 if (ri->slave_master_port != slave_master_port) {
	1889                     ri->slave_master_port = slave_master_port;
	1890                     ri->slave_conf_change_time = mstime();
	1891                 }
	1892             }
	1893
	1894             /* master_link_status:<status> */
	1895             if (sdslen(l) >= 19 && !memcmp(l,"master_link_status:",19)) {
	1896                 ri->slave_master_link_status =
	1897                     (strcasecmp(l+19,"up") == 0) ?
	1898                     SENTINEL_MASTER_LINK_STATUS_UP :
	1899                     SENTINEL_MASTER_LINK_STATUS_DOWN;
	1900             }
	1901
	1902             /* slave_priority:<priority> */
	1903             if (sdslen(l) >= 15 && !memcmp(l,"slave_priority:",15))
	1904                 ri->slave_priority = atoi(l+15);
	1905
	1906             /* slave_repl_offset:<offset> */
	1907             if (sdslen(l) >= 18 && !memcmp(l,"slave_repl_offset:",18))
	1908                 ri->slave_repl_offset = strtoull(l+18,NULL,10);
	1909         }
	1910     }
	1918     /* Remember when the role changed. */
	1919     if (role != ri->role_reported) {
	1920         ri->role_reported_time = mstime();
	1921         ri->role_reported = role;
	1922         if (role == SRI_SLAVE) ri->slave_conf_change_time = mstime();
	1923         /* Log the event with +role-change if the new role is coherent or
	1924          * with -role-change if there is a mismatch with the current config. */
	1925         sentinelEvent(REDIS_VERBOSE,
	1926             ((ri->flags & (SRI_MASTER|SRI_SLAVE)) == role) ?
	1927             "+role-change" : "-role-change",
	1928             ri, "%@ new reported role is %s",
	1929             role == SRI_MASTER ? "master" : "slave",
	1930             ri->flags & SRI_MASTER ? "master" : "slave");
	1931     }
	```
	这部分是针对info中的role章节的，针对slave解析了ri->slave_master_host，ri->slave_master_port，ri->slave_master_link_status，ri->slave_priority，ri->slave_repl_offset这项几个信息出来，并且记录了ri->slave_master_host，ri->slave_master_port这两个值变动的时间到ri->slave_conf_change_time。
	
	- ri->slave_master_host 就是slave sentinelRedisInstance用来记录指向的redis instance认为从info中暴露出来的master_host信息的。ri->slave_master_port同理。如果slave sentinelRedisInstance被转换为master sentinelRedisInstance，ri->slave_master_host会被置为NULL。
	
	- ri->slave_conf_change_time 值得注意的几点是，有且仅有以下几点，
		- 记录ri->slave_master_host变更时间
		
		- 记录ri->slave_master_port变更时间
		
		- 记录该sentinelRedisInstance之前的role为master，现在从指向instance的info中报告是slave的变更时间。
		
		- 所有role的sentinelRedisInstance的这个属性在初始化的时候被初始为初始当时的时间。
	
	- ri->slave_master_link_status 是slave sentinelRedisInstance用来记录从info中暴露出来的slave和其master的master_link_status状态信息。关于master_link_status后续章节会详细介绍。
	
	- ri->slave_priority 是slave sentinelRedisInstance用来记录从info中暴露出来的slave_priority信息。
	
	- ri->slave_repl_offset 是slave sentinelRedisInstance用来记录从info中暴露出来的slave_repl_offset信息。
	
	ri->slave_priority，ri->slave_repl_offset是failover发生时选择好的slave的考虑因素。
	
	这的注意的是ri->flags除了SRI_RECONF_xxx系列之外的其他flag位包括SRI_MASTER,SRI_SLAVE并不会直接被从info得到的信息更新。SRI_RECONF_xxx系列后续会详细介绍。所以可以看到"+role-change"、"-role-change"这两种message的区别是，"+role-change"是flags已经在之前被更新，新来的info信息报告吻合了之前的flags更改。"-role-change"则是新报告的info信息和现有的flags不吻合。

	- ri->role_reported, ri->role_reported_time
		
		- role_reported被初始化为该sentinelRedisInstance创建的role, ri->role_reported = ri->flags & (SRI_MASTER|SRI_SLAVE); 如果role是sentinel，则此flags所有位都是0.
		
		- 当该sentinelRedisInstance被reset 成master角色被重置时，会将ri->role_reported = SRI_MASTER; sentinelResetMaster的细节后续会详细介绍。
		
		- 当然也会用来记录，已经记录的role信息同info信息中暴露的信息不同时的变更。
		
	上面sentinelRefreshInstanceInfo讲的是被动接受记录info信息部分。
	
	下面讲一些根据info信息主观判断参与的部分。
	
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
	1953             /* Now that we are sure the slave was reconfigured as a master
	1954              * set the master configuration epoch to the epoch we won the
	1955              * election to perform this failover. This will force the other
	1956              * Sentinels to update their config (assuming there is not
	1957              * a newer one already available). */
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
	1968         } else {
	1969             /* A slave turned into a master. We want to force our view and
	1970              * reconfigure as slave. Wait some time after the change before
	1971              * going forward, to receive new configs if any. */
	1972             mstime_t wait_time = SENTINEL_PUBLISH_PERIOD*4;
	1973
	1974             if (!(ri->flags & SRI_PROMOTED) &&
	1975                  sentinelMasterLooksSane(ri->master) &&
	1976                  sentinelRedisInstanceNoDownFor(ri,wait_time) &&
	1977                  mstime() - ri->role_reported_time > wait_time)
	1978             {
	1979                 int retval = sentinelSendSlaveOf(ri,
	1980                         ri->master->addr->ip,
	1981                         ri->master->addr->port);
	1982                 if (retval == REDIS_OK)
	1983                     sentinelEvent(REDIS_NOTICE,"+convert-to-slave",ri,"%@");
	1984             }
	1985         }
	1986     }
	```
	Handle slave -> master role switch. 在当前的ri是role为slave的sentinelRedisInstance，而指向的redis instance确报告自己为master这样的情况下，有以下几部分逻辑，
	
	- 如果此时从我们记录的状态来看，该slave sentinelRedisInstance确实是被标记为SRI_PROMOTED，并且ri->master->failover_state为SENTINEL_FAILOVER_STATE_WAIT_PROMOTION，表示failover正在进行中，并且此slave sentinelRedisInstance就是被promote为master的。则会完成首先在此完成提升的这一步的config变更，关于failover和这个config变更的细节后续会详细讨论。
	
	- 如果不是上面的情况, 如果以下几个条件同时满足（mstime_t wait_time = SENTINEL_PUBLISH_PERIOD*4; 8s)
	
		- 如果该slave sentinelRedisInstance没有被我们标记为SRI_PROMOTED,这里稍微岔开以下，提一下关于SRI_PROMOTED，另外几个值得注意的点有，
		
			- 会在回收sentinelRedisInstance处理SRI_PROMOTED状态的slave的master的promoted_slave属性。
			
			```
			/* src/sentinel.c */
			997 /* Release this instance and all its slaves, sentinels, hiredis connections.
			998  * This function does not take care of unlinking the instance from the main
			999  * masters table (if it is a master) or from its master sentinels/slaves table
			1000  * if it is a slave or sentinel. */
			1001 void releaseSentinelRedisInstance(sentinelRedisInstance *ri) {
			1021     /* Clear state into the master if needed. */
			1022     if ((ri->flags & SRI_SLAVE) && (ri->flags & SRI_PROMOTED) && ri->master)
			1023         ri->master->promoted_slave = NULL;
			```
			
			- 会在failover的过程中，被选中的slave sentinelRedisInstance会被置SRI_PROMOTED状态，并且将master sentinelRedisInstance的promoted_slave记录为此slave sentinelRedisInstance。
			
			```			
			3668 void sentinelFailoverSelectSlave(sentinelRedisInstance *ri) {
			3669     sentinelRedisInstance *slave = sentinelSelectSlave(ri);
			3670
			3671     /* We don't handle the timeout in this state as the function aborts
			3672      * the failover or go forward in the next state. */
			3673     if (slave == NULL) {
			3677     } else {
			3678         sentinelEvent(REDIS_WARNING,"+selected-slave",slave,"%@");
			3679         slave->flags |= SRI_PROMOTED;
			3680         ri->promoted_slave = slave;
			3685     }
			3686 }
			```
			
			- 值得注意的是，ri->promoted_slave以及SRI_PROMOTED，只有进行某个master的failiover的sentinel的该master struct里才会记录的相关状态，这两个属性不会传播给其他sentinel。后续还会提到SRI_PROMOTED用于在sentinelFailoverDetectEnd和sentinelFailoverReconfNextSlave中skip掉某些逻辑的用途。
			
		- sentinelMasterLooksSane(ri->master)
		
		```
		/* src/sentinel.c */
		1776 /* Return true if master looks "sane", that is:
		1777  * 1) It is actually a master in the current configuration.
		1778  * 2) It reports itself as a master.
		1779  * 3) It is not SDOWN or ODOWN.
		1780  * 4) We obtained last INFO no more than two times the INFO period time ago. */
		1781 int sentinelMasterLooksSane(sentinelRedisInstance *master) {
		1782     return
		1783         master->flags & SRI_MASTER &&
		1784         master->role_reported == SRI_MASTER &&
		1785         (master->flags & (SRI_S_DOWN|SRI_O_DOWN)) == 0 &&
		1786         (mstime() - master->info_refresh) < SENTINEL_INFO_PERIOD*2;
		1787 }
		```
		
		如果slave sentinelRedisInstance指向master sentinelRedisInstance, 该master sentinelRedisInstance标记自己是SRI_MASTER并且记录的role_reported也是SRI_MASTER，并且没有处于sdown，odown状态，并且距离上一次info更新时间在两个SENTINEL_INFO_PERIOD之内，即20s以内。则认为master看起来不错。
					
		- sentinelRedisInstanceNoDownFor(ri,wait_time)
		
		```
		/* src/sentinel.c */
		1286 /* Return non-zero if there was no SDOWN or ODOWN error associated to this
		1287  * instance in the latest 'ms' milliseconds. */
		1288 int sentinelRedisInstanceNoDownFor(sentinelRedisInstance *ri, mstime_t ms) {
		1289     mstime_t most_recent;
		1290
		1291     most_recent = ri->s_down_since_time;
		1292     if (ri->o_down_since_time > most_recent)
		1293         most_recent = ri->o_down_since_time;
		1294     return most_recent == 0 || (mstime() - most_recent) > ms;
		1295 }
		```
		如果距离master sentinelRedisInstance记录的最近一次down掉的时间大于wait_time 8s的话，则为true
		
		- mstime() - ri->role_reported_time > wait_time
		
		距离上一次role_reported的时间也得大于wait_time，即这个role_reported已经持续了这么长时间，同上面一项合作起来的意思是想表明该master sentinelRedisInstance在wait_time这段时间内没有down过，并且role_reported持续了wait_time没有变过。
		
		上面几个条件完全满足的话，此时会尝试发送sentinelSendSlaveOf纠正这个info中报告自己是master的instance SlaveOf 我们config中的master，以配合sentinel的config记录的那样。并且输出"+convert-to-slave"这样的message，我个人认为要尽量避免这个message出现。
		
	继续讲sentinelRefreshInstanceInfo的剩余部分，
	
	```
	/* src/sentinel.c */
	1789 /* Process the INFO output from masters. */
	1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
	1988     /* Handle slaves replicating to a different master address. */
  	1989     if ((ri->flags & SRI_SLAVE) &&
  	1990         role == SRI_SLAVE &&
  	1991         (ri->slave_master_port != ri->master->addr->port ||
	1992          strcasecmp(ri->slave_master_host,ri->master->addr->ip)))
	1993     {
	1994         mstime_t wait_time = ri->master->failover_timeout;
	1995
	1996         /* Make sure the master is sane before reconfiguring this instance
	1997          * into a slave. */
	1998         if (sentinelMasterLooksSane(ri->master) &&
	1999             sentinelRedisInstanceNoDownFor(ri,wait_time) &&
	2000             mstime() - ri->slave_conf_change_time > wait_time)
	2001         {
	2002             int retval = sentinelSendSlaveOf(ri,
	2003                     ri->master->addr->ip,
	2004                     ri->master->addr->port);
	2005             if (retval == REDIS_OK)
	2006                 sentinelEvent(REDIS_NOTICE,"+fix-slave-config",ri,"%@");
	2007         }
	2008     }
	```
	这段是处理slave replicating to a different master的情况,即如果保存当前slave sentinelRedisInstance的flags信息和从info获取的role信息都表示所指向的instance是slave，但是从info里获取的master信息不吻合，并且满足以下条件，（mstime_t wait_time = ri->master->failover_timeout;）
	
	- sentinelMasterLooksSane(ri->master) maser看起来okay
	
	- sentinelRedisInstanceNoDownFor(ri,wait_time) 这个slave在当前时间往前failover_timeout一段时间之内没down掉过。
	
	- mstime() - ri->slave_conf_change_time > wait_time 在wait_time这段时间之内slave的config没有变更过。(具体变更情况，前面已经列举过了)。
	
	就会强制sentinelSendSlaveOf这个slave instance SlaveOf 我们config中的master instance。
	
	sentinelRefreshInstanceInfo最后一部分，
	
	```
	/* src/sentinel.c */
	1789 /* Process the INFO output from masters. */
	1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
	2010     /* Detect if the slave that is in the process of being reconfigured
	2011      * changed state. */
	2012     if ((ri->flags & SRI_SLAVE) && role == SRI_SLAVE &&
	2013         (ri->flags & (SRI_RECONF_SENT|SRI_RECONF_INPROG)))
	2014     {
	2015         /* SRI_RECONF_SENT -> SRI_RECONF_INPROG. */
	2016         if ((ri->flags & SRI_RECONF_SENT) &&
	2017             ri->slave_master_host &&
	2018             strcmp(ri->slave_master_host,
	2019                     ri->master->promoted_slave->addr->ip) == 0 &&
	2020             ri->slave_master_port == ri->master->promoted_slave->addr->port)
	2021         {
	2022             ri->flags &= ~SRI_RECONF_SENT;
	2023             ri->flags |= SRI_RECONF_INPROG;
	2024             sentinelEvent(REDIS_NOTICE,"+slave-reconf-inprog",ri,"%@");
	2025         }
	2026
	2027         /* SRI_RECONF_INPROG -> SRI_RECONF_DONE */
	2028         if ((ri->flags & SRI_RECONF_INPROG) &&
	2029             ri->slave_master_link_status == SENTINEL_MASTER_LINK_STATUS_UP)
	2030         {
	2031             ri->flags &= ~SRI_RECONF_INPROG;
	2032             ri->flags |= SRI_RECONF_DONE;
	2033             sentinelEvent(REDIS_NOTICE,"+slave-reconf-done",ri,"%@");
	2034         }
	2035     }
	```
	这是failover过程中SRI_RECONF_xx这个系列相关的逻辑,这个逻辑是发生在role为slave的sentinelRedisInstance上的，这系列flag的定义如下：
	
	```
	/* src/sentinel.c */
	65 #define SRI_RECONF_SENT (1<<9)     /* SLAVEOF <newmaster> sent. */
  	66 #define SRI_RECONF_INPROG (1<<10)   /* Slave synchronization in progress. */
  	67 #define SRI_RECONF_DONE (1<<11)     /* Slave synchronized with new master. */
	```
	如果该instance的info中报告的该instance的role是slave，并且slave sentinelRedisInstance的flags记录表示该instance正处于前两个状态，即SRI_RECONF_SENT，SRI_RECONF_INPROG，则需要检查更多的信息来判断这系列状态是否要往后推进，包括以下两个阶段。
	
	- SRI_RECONF_SENT -> SRI_RECONF_INPROG
	如果该slave sentinelRedisInstance处于SRI_RECONF_SENT状态，并且该slave sentinelRedisInstance指向的slave instance的info中报告的ri->slave_master_host和ri->slave_master_port和该slave sentinelRedisInstance记录的ri->master->promoted_slave的host和port相吻合，则说明在sendslaveof cmd发出去之后，instance已经响应slave of完成，则标记flags为SRI_RECONF_INPROG进入到下一个阶段。
	
	- SRI_RECONF_INPROG -> SRI_RECONF_DONE
	如果该slave sentinelRedisInstance处于SRI_RECONF_SENT_INPROG状态，并且相应的slave instance的info中报告的ri->slave_master_link_status已经处于up状态，则认为整个reconf完成。关于master_link_status具体的含义后续会详细解释。
	
	另外SRI_RECONF_xx系列还有以下几个用途，
	
	- 主导failover过程的sentinel判断该次failover是否结束时，会判断处于该次failover的那个master sentinelRedisInstance挂载下的所有slave sentinelRedisInstance是否处于SRI_RECONF_DONE的状态，或者是SRI_PROMOTED状态，SRI_PROMOTED是因为该slave sentinelRedisInstance相应的slave instance就是这次failover过程中被选出来当做新的master的instance，不需要经历SRI_RECONF_xx系列。
	
	```
	/* src/sentinel.c */
	3729 void sentinelFailoverDetectEnd(sentinelRedisInstance *master) {
	3740     /* The failover terminates once all the reachable slaves are properly
	3741      * configured. */
	3742     di = dictGetIterator(master->slaves);
	3743     while((de = dictNext(di)) != NULL) {
	3744         sentinelRedisInstance *slave = dictGetVal(de);
	3745
	3746         if (slave->flags & (SRI_PROMOTED|SRI_RECONF_DONE)) continue;
	3747         if (slave->flags & SRI_S_DOWN) continue;
	3748         not_reconfigured++;
	3749     }
	3750     dictReleaseIterator(di);
	```
	
	- SRI_RECONF_xx系列 在sentinelFailoverReconfNextSlave的详细作用后续会详细说明。


	上面那么大的篇幅终于把info信息的处理这部分内容讲完了，除了sentinelInfoReplyCallback这个定期收集info信息的逻辑之外，还有sentinelSendPing也是定期执行的。
	
	```
	/* src/sentinel.c */
	2342 /* Send periodic PING, INFO, and PUBLISH to the Hello channel to
	2343  * the specified master or slave instance. */
	2344 void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {
	2386     } else if ((now - ri->last_pong_time) > ping_period) {
	2387         /* Send PING to all the three kinds of instances. */
	2388         sentinelSendPing(ri);
	
	2322 /* Send a PING to the specified instance and refresh the last_ping_time
	2323  * if it is zero (that is, if we received a pong for the previous ping).
	2324  *
	2325  * On error zero is returned, and we can't consider the PING command
	2326  * queued in the connection. */
	2327 int sentinelSendPing(sentinelRedisInstance *ri) {
	2328     int retval = redisAsyncCommand(ri->cc,
	2329         sentinelPingReplyCallback, NULL, "PING");
	2330     if (retval == REDIS_OK) {
	2331         ri->pending_commands++;
	2332         /* We update the ping time only if we received the pong for
	2333          * the previous ping, otherwise we are technically waiting
	2334          * since the first ping that did not received a reply. */
	2335         if (ri->last_ping_time == 0) ri->last_ping_time = mstime();
	2336         return 1;
	2337     } else {
	2338         return 0;
	2339     }
	2340 }
	```
	
	可以看出来，ping这个cmd是发给master,slave,sentinel三种role的sentinelRedisInstance所指向的instance的，还可以看到该sentinelRedisInstance的ri->last_ping_time是在ri->last_ping_time为0这个初始状态才会更新的。last_ping_time和last_pong_time之间的恩怨如下。
		
	- ri->last_ping_time, ri->last_pong_time初始化为初始当时时间
	
	```
	/* src/sentinel.c */
	896 sentinelRedisInstance *createSentinelRedisInstance(char *name, int flags, char *hostname, int port, int quorum, sentinelRedisInstance *master) {
	944     /* We set the last_ping_time to "now" even if we actually don't have yet
 	945      * a connection with the node, nor we sent a ping.
 	946      * This is useful to detect a timeout in case we'll not be able to connect
 	947      * with the node at all. */
	948     ri->last_ping_time = mstime();
 	950     ri->last_pong_time = mstime();
 	
 	1164 #define SENTINEL_RESET_NO_SENTINELS (1<<0)
	1165 void sentinelResetMaster(sentinelRedisInstance *ri, int flags) {
	1188     ri->last_ping_time = mstime();
	1189     ri->last_avail_time = mstime();
	1190     ri->last_pong_time = mstime();
	```
	
	- 即ping cmd会在ri->last_pong_time最后更新距今已经大于一个ping_period间隔时间持续进行。
	
	```
	/* src/sentinel.c */
	2342 /* Send periodic PING, INFO, and PUBLISH to the Hello channel to
	2343  * the specified master or slave instance. */
	2344 void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {
	2386     } else if ((now - ri->last_pong_time) > ping_period) {
	2387         /* Send PING to all the three kinds of instances. */
	2388         sentinelSendPing(ri);
	```
	
	- 如果得到了acceptable reply，则更新ri->last_ping_time = 0;作为收到了acceptable reply信息的标记。值得注意的是收到任何reply即使没有收到acceptable reply，都会更新ri->last_pong_time,会影响上面一条的行为。
	
	```
	/* src/sentinel.c */
	2062 void sentinelPingReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {
	2063     sentinelRedisInstance *ri = c->data;
	2064     redisReply *r;
	2065     REDIS_NOTUSED(privdata);
	2066
	2067     if (ri) ri->pending_commands--;
	2068     if (!reply || !ri) return;
	2069     r = reply;
	2071     if (r->type == REDIS_REPLY_STATUS ||
	2072         r->type == REDIS_REPLY_ERROR) {
	2073         /* Update the "instance available" field only if this is an
	2074          * acceptable reply. */
	2075         if (strncmp(r->str,"PONG",4) == 0 ||
	2076             strncmp(r->str,"LOADING",7) == 0 ||
	2077             strncmp(r->str,"MASTERDOWN",10) == 0)
	2078         {
	2079             ri->last_avail_time = mstime();
	2080             ri->last_ping_time = 0; /* Flag the pong as received. */
	2096     ri->last_pong_time = mstime();
	```

	- send ping cmd,如果此时ri->last_ping_time为0，这是一个从之前的ping获得了acceptable reply的标记，则再次更新ri->last_ping_time.表示获得acceptable reply之后最早一次pending的ping是什么时候发出去的。同时不为0也表示至少有一个ping cmd还在pending中，未获得回应。
	
	```
	/* src/sentinel.c */
	2322 /* Send a PING to the specified instance and refresh the last_ping_time
	2323  * if it is zero (that is, if we received a pong for the previous ping).
	2324  *
	2325  * On error zero is returned, and we can't consider the PING command
	2326  * queued in the connection. */
	2327 int sentinelSendPing(sentinelRedisInstance *ri) {
	2328     int retval = redisAsyncCommand(ri->cc,
	2329         sentinelPingReplyCallback, NULL, "PING");
	2330     if (retval == REDIS_OK) {
	2331         ri->pending_commands++;
	2332         /* We update the ping time only if we received the pong for
	2333          * the previous ping, otherwise we are technically waiting
	2334          * since the first ping that did not received a reply. */
	2335         if (ri->last_ping_time == 0) ri->last_ping_time = mstime();
	2336         return 1;
	2337     } else {
	2338         return 0;
	2339     }
	2340 }
	```
	
	上面的ping pong恩怨占了sentinelPing的一部分逻辑，因为ri->last_ping_time，ri->last_pong_timer的更新并不仅是记录作用，ri->last_ping_time被用作sentinelCheckSubjectivelyDown的检查逻辑,具体细节后续会详细介绍。
	
	sentinelPingReplyCallback的另外的逻辑
	
	```
	/* src/sentinel.c */
	2062 void sentinelPingReplyCallback(redisAsyncContext *c, void *reply, void *privdata) {
	2063     sentinelRedisInstance *ri = c->data;
	2064     redisReply *r;
	2065     REDIS_NOTUSED(privdata);
	2066
	2067     if (ri) ri->pending_commands--;
	2068     if (!reply || !ri) return;
	2069     r = reply;
	2070
	2071     if (r->type == REDIS_REPLY_STATUS ||
	2072         r->type == REDIS_REPLY_ERROR) {
	2073         /* Update the "instance available" field only if this is an
	2074          * acceptable reply. */
	2075         if (strncmp(r->str,"PONG",4) == 0 ||
	2076             strncmp(r->str,"LOADING",7) == 0 ||
	2077             strncmp(r->str,"MASTERDOWN",10) == 0)
	2078         {
	2079             ri->last_avail_time = mstime();
	2080             ri->last_ping_time = 0; /* Flag the pong as received. */
	2081         } else {
	2082             /* Send a SCRIPT KILL command if the instance appears to be
	2083              * down because of a busy script. */
	2084             if (strncmp(r->str,"BUSY",4) == 0 &&
	2085                 (ri->flags & SRI_S_DOWN) &&
	2086                 !(ri->flags & SRI_SCRIPT_KILL_SENT))
	2087             {
	2088                 if (redisAsyncCommand(ri->cc,
	2089                         sentinelDiscardReplyCallback, NULL,
	2090                         "SCRIPT KILL") == REDIS_OK)
	2091                     ri->pending_commands++;
	2092                 ri->flags |= SRI_SCRIPT_KILL_SENT;
	2093             }
	2094         }
	2095     }
	2096     ri->last_pong_time = mstime();
	2097 }
	```
		
	- 在	收到acceptable reply时，不仅更新ri->last_ping_time = 0;，并且更新了ri->last_avail_time，这个属性和ri->last_pong_time的不同区别就在于尽在acceptable reply时才更新ri->last_avail_time。这也是ri->last_avail_time的全部意义。
	
	- Send a SCRIPT KILL command if the instance appears to be down because of a busy script. 这也是SRI_SCRIPT_KILL_SENT的全部用途。
