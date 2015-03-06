## **sentinel failover过程的详细介绍**
------------------------------------

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

### **sentinel monitor常态的信息收集**
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
	
	从之前介绍的sentinel的拓扑结构结合此处来看，每个master sentinelRedisInstance struct会创建一个cc连接和一个pc连接到指向的相应的redis instance，每个master sentinelRedisInstance struct的\*slaves dict所指向的所有slave sentinelRedisInstance struct也会一一创建一个cc连接和一个pc连接到指向的相应的redis instance。另外对于每个master sentinelRedisInstance struct的\*sentinels dict所指向的所有sentinel sentinelRedisInstance也会一一创建一个cc连接到指向的sentinel instance（没有pc，sentinel不会直接订阅其他的sentinels）。

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
	
- sentinelSendPeriodicCommands



### **发现异常sdown，odown**
--------------------------
这一章节介绍sentinelHandleRedisInstance里的另外两个函数

- sentinelCheckSubjectivelyDown

- sentinelCheckObjectivelyDown

-  

### **vote for leader & try failover**
--------------------------------------

### **wait start failover**
---------------------------

### **failover end success**
----------------------------

### **failover end fail**
----------------------------





