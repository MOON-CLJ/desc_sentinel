### **发现异常sdown，odown**
--------------------------
这一章节介绍sentinelHandleRedisInstance里的另外两个函数

```
/* src/sentinel.c */
3918 /* Perform scheduled operations for the specified Redis instance. */
3919 void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {
3935     /* Every kind of instance */
3936     sentinelCheckSubjectivelyDown(ri);
3943     /* Only masters */
3944     if (ri->flags & SRI_MASTER) {
3945         sentinelCheckObjectivelyDown(ri);
```
可以看出来sentinelCheckSubjectivelyDown可以作用于master,slave,sentinel任意一种role的sentinelRedisInstance上，而sentinelCheckObjectivelyDown仅能作用于master role的sentinelRedisInstance上。

- sentinelCheckSubjectivelyDown

```
/* src/sentinel.c */
3044 /* Is this instance down from our point of view? */
3045 void sentinelCheckSubjectivelyDown(sentinelRedisInstance *ri) {
3046     mstime_t elapsed = 0;
3047
3048     if (ri->last_ping_time)
3049         elapsed = mstime() - ri->last_ping_time;
3050
3051     /* Check if we are in need for a reconnection of one of the
3052      * links, because we are detecting low activity.
3053      *
3054      * 1) Check if the command link seems connected, was connected not less
3055      *    than SENTINEL_MIN_LINK_RECONNECT_PERIOD, but still we have a
3056      *    pending ping for more than half the timeout. */
3057     if (ri->cc &&
3058         (mstime() - ri->cc_conn_time) > SENTINEL_MIN_LINK_RECONNECT_PERIOD &&
3059         ri->last_ping_time != 0 && /* Ther is a pending ping... */
3060         /* The pending ping is delayed, and we did not received
3061          * error replies as well. */
3062         (mstime() - ri->last_ping_time) > (ri->down_after_period/2) &&
3063         (mstime() - ri->last_pong_time) > (ri->down_after_period/2))
3064     {
3065         sentinelKillLink(ri,ri->cc);
3066     }
3067
3068     /* 2) Check if the pubsub link seems connected, was connected not less
3069      *    than SENTINEL_MIN_LINK_RECONNECT_PERIOD, but still we have no
3070      *    activity in the Pub/Sub channel for more than
3071      *    SENTINEL_PUBLISH_PERIOD * 3.
3072      */
3073     if (ri->pc &&
3074         (mstime() - ri->pc_conn_time) > SENTINEL_MIN_LINK_RECONNECT_PERIOD &&
3075         (mstime() - ri->pc_last_activity) > (SENTINEL_PUBLISH_PERIOD*3))
3076     {
3077         sentinelKillLink(ri,ri->pc);
3078     }
```

sentinelCheckSubjectivelyDown的前半部分主要是kill link以供重连的逻辑。

检查command link需要重连：

	- 如果距离上次试图connect的ri->cc_conn_time已经过了SENTINEL_MIN_LINK_RECONNECT_PERIOD,SENTINEL_MIN_LINK_RECONNECT_PERIOD默认为15s，也仅仅用于此作用。
	
	- 并且ri->last_ping_time不为0
	
	- ri->last_ping_time在ri->down_after_period/2时间内没有被更新过了
	
	- ri->last_pong_time在ri->down_after_period/2时间内没有被更新过了，即在ri->down_after_period/2没有任何reply。

所以可以看出来ri->down_after_period会严重影响cc重连的行为,ri->down_after_period的行为影响也仅仅是在sentinelCheckSubjectivelyDown里, 除了

- 对ping_period=min(ri->down_after_period, SENTINEL_PING_PERIOD)这样一个小影响之外。

- 在sentinelSelectSlave里max_master_down_time += master->down_after_period * 10

检查pubsub link需要重连：

- 如果距离上次试图connect的ri->pc_conn_time已经过了SENTINEL_MIN_LINK_RECONNECT_PERIOD.

- mstime() - ri->pc_last_activity) > (SENTINEL_PUBLISH_PERIOD*3) 关于ri->pc_last_activity，后续会详细解释.

ri->down_after_period的含义如下：

	- 初始化为SENTINEL_DEFAULT_DOWN_AFTER,即30s，并且会默认以master sentinelRedisInstance struct为准扩散到master->slaves, master->sentinels。
	
	```
	/* src/sentinel.c */
	896 sentinelRedisInstance *createSentinelRedisInstance(char *name, int flags, char *hostname, int port, int quorum, sentinelRedisInstance *master) {
	956     ri->down_after_period = master ? master->down_after_period :
    957                                 SENTINEL_DEFAULT_DOWN_AFTER;
    
    1315 /* This function sets the down_after_period field value in 'master' to all
	1316  * the slaves and sentinel instances connected to this master. */
	1317 void sentinelPropagateDownAfterPeriod(sentinelRedisInstance *master) {
	1318     dictIterator *di;
	1319     dictEntry *de;
	1320     int j;
	1321     dict *d[] = {master->slaves, master->sentinels, NULL};
	1322
	1323     for (j = 0; d[j]; j++) {
	1324         di = dictGetIterator(d[j]);
	1325         while((de = dictNext(di)) != NULL) {
	1326             sentinelRedisInstance *ri = dictGetVal(de);
	1327             ri->down_after_period = master->down_after_period;
	1328         }
	1329         dictReleaseIterator(di);
	1330     }
  	1331 }
	```
	- 在sentinelCheckSubjectivelyDown的作用马上后详细解释。
	
```
/* src/sentinel.c */
3044 /* Is this instance down from our point of view? */
3045 void sentinelCheckSubjectivelyDown(sentinelRedisInstance *ri) {
3046     mstime_t elapsed = 0;
3047
3048     if (ri->last_ping_time)
3049         elapsed = mstime() - ri->last_ping_time;
3080     /* Update the SDOWN flag. We believe the instance is SDOWN if:
3081      *
3082      * 1) It is not replying.
3083      * 2) We believe it is a master, it reports to be a slave for enough time
3084      *    to meet the down_after_period, plus enough time to get two times
3085      *    INFO report from the instance. */
3086     if (elapsed > ri->down_after_period ||
3087         (ri->flags & SRI_MASTER &&
3088          ri->role_reported == SRI_SLAVE &&
3089          mstime() - ri->role_reported_time >
3090           (ri->down_after_period+SENTINEL_INFO_PERIOD*2)))
3091     {
3092         /* Is subjectively down */
3093         if ((ri->flags & SRI_S_DOWN) == 0) {
3094             sentinelEvent(REDIS_WARNING,"+sdown",ri,"%@");
3095             ri->s_down_since_time = mstime();
3096             ri->flags |= SRI_S_DOWN;
3097         }
3098     } else {
3099         /* Is subjectively up */
3100         if (ri->flags & SRI_S_DOWN) {
3101             sentinelEvent(REDIS_WARNING,"-sdown",ri,"%@");
3102             ri->flags &= ~(SRI_S_DOWN|SRI_SCRIPT_KILL_SENT);
3103         }
3104     }
3105 }
```	

sentinelCheckSubjectivelyDown的后半部就是认定或者取消SRI_S_DOWN的状态。

- +sdown 
	
	- mstime() - ri->last_ping_time > ri->down_after_period, 即如果ri->last_ping_time不为0，但是是pending状态即没有获得acceptable reply已经持续了超过ri->down_after_period.ri->last_ping_time在sentinelCheckSubjectivelyDown中的作用就在于此.
		
	- 如果该sentinelRedisInstance的ri->flags记录的role是master，而ri->role_reported报告是slave，并且报告已经超过(ri->down_after_period+SENTINEL_INFO_PERIOD*2)默认为50s的时间。此种情况下也会触发+sdown，为强制failover当前config中记录的该master创造条件。ri->down_after_period在在sentinelCheckSubjectivelyDown中的作用就在于此.
		
	如果以上条件满足，则会检查SRI_S_DOWN并标记flags为SRI_S_DOWN状态，并更新ri->s_down_since_time。并输出+sdown message。
	
- -sdown

	如果+sdown的条件不满足，则检查SRI_S_DOWN并撤销SRI_S_DOWN状态，并输出-sdown标记。

SRI_S_DOWN标记只在以上两种情况下更新，也就是说这两个状态之间是来回切换的，不会有连续两次认定SRI_S_DOWN状态，也不会连续两次撤消SRI_S_DOWN状态。

- sentinelCheckObjectivelyDown

```
/* src/sentinel.c */
3107 /* Is this instance down according to the configured quorum?
3108  *
3109  * Note that ODOWN is a weak quorum, it only means that enough Sentinels
3110  * reported in a given time range that the instance was not reachable.
3111  * However messages can be delayed so there are no strong guarantees about
3112  * N instances agreeing at the same time about the down state. */
3113 void sentinelCheckObjectivelyDown(sentinelRedisInstance *master) {
3114     dictIterator *di;
3115     dictEntry *de;
3116     unsigned int quorum = 0, odown = 0;
3117
3118     if (master->flags & SRI_S_DOWN) {
3119         /* Is down for enough sentinels? */
3120         quorum = 1; /* the current sentinel. */
3121         /* Count all the other sentinels. */
3122         di = dictGetIterator(master->sentinels);
3123         while((de = dictNext(di)) != NULL) {
3124             sentinelRedisInstance *ri = dictGetVal(de);
3125
3126             if (ri->flags & SRI_MASTER_DOWN) quorum++;
3127         }
3128         dictReleaseIterator(di);
3129         if (quorum >= master->quorum) odown = 1;
3130     }
3131
3132     /* Set the flag accordingly to the outcome. */
3133     if (odown) {
3134         if ((master->flags & SRI_O_DOWN) == 0) {
3135             sentinelEvent(REDIS_WARNING,"+odown",master,"%@ #quorum %d/%d",
3136                 quorum, master->quorum);
3137             master->flags |= SRI_O_DOWN;
3138             master->o_down_since_time = mstime();
3139         }
3140     } else {
3141         if (master->flags & SRI_O_DOWN) {
3142             sentinelEvent(REDIS_WARNING,"-odown",master,"%@");
3143             master->flags &= ~SRI_O_DOWN;
3144         }
3145     }
3146 }
```

可以看到认定为odown有两个条件，

- 首先该master sentinelRedisInstance处于SRI_S_DOWN状态下，
	
- 并且统计该master sentinelRedisInstance挂载下的sentinel sentinelRedisInstance，大部分sentinel sentinelRedisInstance处于SRI_MASTER_DOWN状态下。可以看到quorum的第一个作用就是在此，用于统计大多数，包括自己在内>=quorum。SRI_MASTER_DOWN这个flag的含义后续会详细介绍。关于quorum其他用途后续还会提到。

结果如下,

- +odown

	如果以上条件满足，则会检查SRI_O_DOWN并标记flags为SRI_O_DOWN状态，并更新ri->o_down_since_time。并输出+odown message。
	
- -down
	反之，则检查SRI_O_DOWN并撤销flags的SRI_O_DOWN状态，并输出-odown message。

SRI_O_DOWN标记只在以上两种情况下更新，也就是说这两个状态之间是来回切换的，不会有连续两次认定SRI_O_DOWN状态，也不会连续两次撤消SRI_O_DOWN状态。

但是值得注意的是，+odown状态对+sdown状态有依赖关系，并且显而易见，满足的条件上面也提过。但是-odown对-sdown的依赖需要小心对待。分两种情况

- 如果该master sentinelRedisInstance的SRI_S_DOWN状态撤消了，则SRI_O_DOWN一定会撤销，不通过任何统计大多数的流程。

- 如果该master sentinelRedisInstance的SRI_S_DOWN状态还在，但是从该master sentinelRedisInstance挂载下的sentinel sentinelRedisInstance中统计SRI_MASTER_DOWN状态没有达到大多数同意时，SRI_O_DOWN还是会撤销。

所以不能假设+sdown,+odown,-sdown,-odown一定是顺序发生的，提这个事情主要是之前假设过这个逻辑，但事实证明假设不成立。当时的日志如下。

> 我又看了一下，确实可能出现+odown之后马上就来一个-odown，中间没有-sdown的情况，+odown之后，此sentinel会立即向其他sentinels发消息确认，14:24:28的时候，独自认为+odown的那个sentinel向其他sentinel发消息确认是不是 +odown了，别人告诉他“没有啊”，那他说，“哦，我搞错了，不好意思”，然后马上就直接-odown了。但是14:24:30此sentinel又从自身的状态统计到了+odown，这次他问其他sentinels，别人都同意.

### **start failover & try failover**
--------------------------------------

这一节介绍将准备进入failover state machine之前的一些步骤，不涉及failover的执行流程, 涉及到投票过程，但本节不会涉及vote过程，关于vote后续会详细解释。

```
/* src/sentinel.c */
3918 /* Perform scheduled operations for the specified Redis instance. */
3919 void sentinelHandleRedisInstance(sentinelRedisInstance *ri) {
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
整个failover的流程还是在sentinelHandleRedisInstance下。sentinelStartFailoverIfNeeded会不断检查failover是否进入sentinelStartFailover的逻辑。

```
/* src/sentinel.c */
3483  * 1) Master must be in ODOWN condition.
3484  * 2) No failover already in progress.
3485  * 3) No failover already attempted recently.
3486  *
3487  * We still don't know if we'll win the election so it is possible that we
3488  * start the failover but that we'll not be able to act.
3489  *
3490  * Return non-zero if a failover was started. */
3491 int sentinelStartFailoverIfNeeded(sentinelRedisInstance *master) {
3492     /* We can't failover if the master is not in O_DOWN state. */
3493     if (!(master->flags & SRI_O_DOWN)) return 0;
3494
3495     /* Failover already in progress? */
3496     if (master->flags & SRI_FAILOVER_IN_PROGRESS) return 0;
3497
3498     mstime_t now = mstime();
3499
3500     /* Last failover attempt started too little time ago? */
3501     if (now - master->failover_start_time <
3502         master->failover_timeout*2)
3503     {
3504         if (master->failover_delay_logged != master->failover_start_time) {
3511             master->failover_delay_logged = master->failover_start_time;
3515         }
3516         return 0;
3517     }
3525     sentinelStartFailover(master);
3526     return 1;
3527 }
```
- master sentinelRedisInstance必须在SRI_O_DOWN状态

- master sentinelRedisInstance的flags表明已经处于SRI_FAILOVER_IN_PROGRESS状态了。SRI_FAILOVER_IN_PROGRESS具体的逻辑后续会讲。

- 距离上次master->failover_start_time更新超过master->failover_timeout*2的时间。failover_start_time的具体逻辑后续会解释。master->failover_delay_logged就是用来在此处记录master->failover_start_time用于输出log的逻辑。

上面三种情况下不满足的情况下返回0，否则则进入sentinelStartFailover的逻辑，并返回1。

```
/* src/sentinel.c */
3459 /* Setup the master state to start a failover. */
3460 void sentinelStartFailover(sentinelRedisInstance *master) {
3461     redisAssert(master->flags & SRI_MASTER);
3462
3463     master->failover_state = SENTINEL_FAILOVER_STATE_WAIT_START;
3464     master->flags |= SRI_FAILOVER_IN_PROGRESS;
3465     master->failover_epoch = ++sentinel.current_epoch;
3466     sentinelEvent(REDIS_WARNING,"+new-epoch",master,"%llu",
3467         (unsigned long long) master->failover_epoch);
3468     sentinelEvent(REDIS_WARNING,"+try-failover",master,"%@ %llu",
3469         (unsigned long long) master->failover_epoch);
3470     mstime_t last_time = master->failover_start_time;
3471     master->failover_start_time = mstime()+rand()%SENTINEL_MAX_DESYNC;
3472     redisLog(REDIS_NOTICE,
3473         "failover_start_time update %llu -> %llu",
3474         last_time,
3475         master->failover_start_time);
3476
3477     master->failover_state_change_time = mstime();
3478 }
```
首选检查了该sentinelRedisInstance struct的role必须是master。然后将master->failover_state = SENTINEL_FAILOVER_STATE_WAIT_START;，这是failover状态机的第一个状态。然后设置master->flags |= SRI_FAILOVER_IN_PROGRESS;SRI_FAILOVER_IN_PROGRESS表示的是该master sentinelRedisInstance struct正在进行failover的状态。然后此处更新master->failover_start_time为当前时间，用来避免在master->failover_timeout*2时间内重复对该master进行failover，这是更新master->failover_start_time的一个地方。并且也更新了master->failover_state_change_time，master->failover_state_change_time后续解释他的用途。另外sentinel.current_epoch，master->failover_epoch两个epoch后续也会统一解释。

如果sentinelStartFailoverIfNeeded返回1，则会进入sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_ASK_FORCED);的逻辑。另外即使没有返回1，对于master role的sentinelRedisInstance，也会常规的进行sentinelAskMasterStateToOtherSentinels(ri,SENTINEL_NO_FLAGS);的逻辑。SENTINEL_NO_FLAGS和SENTINEL_ASK_FORCED的区别只是用来控制紧急情况下的ask频率而已,现在这两个flag的用途也仅限于此。

```
/* src/sentinel.c */
3188 /* If we think the master is down, we start sending
3189  * SENTINEL IS-MASTER-DOWN-BY-ADDR requests to other sentinels
3190  * in order to get the replies that allow to reach the quorum
3191  * needed to mark the master in ODOWN state and trigger a failover. */
3192 #define SENTINEL_ASK_FORCED (1<<0)
3193 void sentinelAskMasterStateToOtherSentinels(sentinelRedisInstance *master, int flags) {
3194     dictIterator *di;
3195     dictEntry *de;
3196
3197     di = dictGetIterator(master->sentinels);
3198     while((de = dictNext(di)) != NULL) {
3199         sentinelRedisInstance *ri = dictGetVal(de);
3200         mstime_t elapsed = mstime() - ri->last_master_down_reply_time;
3201         char port[32];
3202         int retval;
3203
3204         /* If the master state from other sentinel is too old, we clear it. */
3205         if (elapsed > SENTINEL_ASK_PERIOD*5) {
3206             ri->flags &= ~SRI_MASTER_DOWN;
3207             sdsfree(ri->leader);
3208             ri->leader = NULL;
3209         }
3210
3211         /* Only ask if master is down to other sentinels if:
3212          *
3213          * 1) We believe it is down, or there is a failover in progress.
3214          * 2) Sentinel is connected.
3215          * 3) We did not received the info within SENTINEL_ASK_PERIOD ms. */
3216         if ((master->flags & SRI_S_DOWN) == 0) continue;
3217         if (ri->flags & SRI_DISCONNECTED) continue;
3218         if (!(flags & SENTINEL_ASK_FORCED) &&
3219             mstime() - ri->last_master_down_reply_time < SENTINEL_ASK_PERIOD)
3220             continue;
3221
3222         /* Ask */
3223         ll2string(port,sizeof(port),master->addr->port);
3224         retval = redisAsyncCommand(ri->cc,
3225                     sentinelReceiveIsMasterDownReply, NULL,
3226                     "SENTINEL is-master-down-by-addr %s %s %llu %s",
3227                     master->addr->ip, port,
3228                     sentinel.current_epoch,
3229                     (master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ?
3230                     server.runid : "*");
3231         if (retval == REDIS_OK) ri->pending_commands++;
3232     }
3233     dictReleaseIterator(di);
3234 }
```
这个函数的逻辑作用于master sentinelRedisInstance上挂载的sentinels sentinelRedisInstance struct，依次向这些sentinel sentinelRedisInstance所指向的sentinel instance发出is-master-down-by-addr cmd。
如果距离上次更新ri->last_master_down_reply_time已经过去SENTINEL_ASK_PERIOD*5,则认为之前记录的SRI_MASTER_DOWN状态以及ri->leader信息过期失效。ri->leader后续会详细解释。SRI_MASTER_DOWN状态马上会介绍。

在以下几种情况下，会跳过执行cmd。

- 如果我们关心的master sentinelRedisInstance的flags表示没有处于SRI_S_DOWN状态，就是sdown都不成立。

- 如果该sentinel sentinelRedisInstance的flags处于SRI_DISCONNECTED的状态，就是该sentinel instance可能处于失联状态。

- 除非SENTINEL_ASK_FORCED为true，否则检查距离上次更新ri->last_master_down_reply_time是否还没超过一个SENTINEL_ASK_PERIOD。

接下来详细介绍一下sentinelReceiveIsMasterDownReply这个回调函数，暂时还是不谈sentinel.current_epoch。

```
/* src/sentinel.c */
3148 /* Receive the SENTINEL is-master-down-by-addr reply, see the
3149  * sentinelAskMasterStateToOtherSentinels() function for more information. */
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
3167         ri->last_master_down_reply_time = mstime();
3168         if (r->element[0]->integer == 1) {
3169             ri->flags |= SRI_MASTER_DOWN;
3170         } else {
3171             ri->flags &= ~SRI_MASTER_DOWN;
3172         }
3173         if (strcmp(r->element[1]->str,"*")) {
3174             /* If the runid in the reply is not "*" the Sentinel actually
3175              * replied with a vote. */
3176             sdsfree(ri->leader);
3177             if ((long long)ri->leader_epoch != r->element[2]->integer)
3178                 redisLog(REDIS_WARNING,
3179                     "%s voted for %s %llu", ri->name,
3180                     r->element[1]->str,
3181                     (unsigned long long) r->element[2]->integer);
3182             ri->leader = sdsnew(r->element[1]->str);
3183             ri->leader_epoch = r->element[2]->integer;
3184         }
3185     }
3186 }
```
可以看到此处，此处针对合法的reply，更新了ri->last_master_down_reply_time，SRI_MASTER_DOWN状态以及ri->leader，ri->leader_epoch。

可以看到ri->last_master_down_reply_time就是作用于此，记录最近一次该sentinel sentinelRedisInstance所指向的sentinel instance对is-master-down-by-addr cmd的回应时间。

第一个参数r->element[0]->integer是就是远程sentinel instance对master down状态的判断，所以直接存入该sentinel sentinelRedisInstance的SRI_MASTER_DOWN flag中。可以看到SRI_MASTER_DOWN的作用也仅仅在于此。

另外可以看到第二个返回参数r->element[1]->str，如果不是返回的"\*"这样的一个字符的话，则表明是一个vote reply。对应的可以看到之前(master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ? server.runid : "\*"请求时的最后一个参数也是有区别的，如果failover_state大于SENTINEL_FAILOVER_STATE_NONE，则表明此时该sentinel正在进行failover急需其他sentinels的投票参与,最后一次请求参数会带上自己的runid，否则则是"*",用于常规性质的统计SRI_MASTER_DOWN状态而已。

关于ri->leader,ri->leader_epoch后续会详细解释，从此处看来，两者的用途是记录sentinel sentinelRedisInstance的投票信息。

看一下sentinel instance对is-master-down-by-addr的reply逻辑，

```
/* src/sentinel.c */
2628 void sentinelCommand(redisClient *c) {
2657     } else if (!strcasecmp(c->argv[1]->ptr,"is-master-down-by-addr")) {
2658         /* SENTINEL IS-MASTER-DOWN-BY-ADDR <ip> <port> <current-epoch> <runid>*/
2659         sentinelRedisInstance *ri;
2660         long long req_epoch;
2661         uint64_t leader_epoch = 0;
2662         char *leader = NULL;
2663         long port;
2664         int isdown = 0;
2665
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
```

- 可以看到首先通过ip，port去sentinel.masters查找相应的master sentinelRedisInstance.

- 如果找不到，则不回应关于down关于vote的任何有效的信息。

- 如果找到，则如果该sentinelRedisInstance是role为master，并且处于SRI_S_DOWN状态，则回复down为1

- 并且针对要求投票的请求，回复投票信息.sentinelVoteLeader的细节后续会详细解释。现在可以稍微注意一下的是，这些投票信息是在master sentinelRedisInstance上或者sentinel这个global struct上的。

### **sentinel failover stateMachine**
--------------------------------------
这章开始，还是需要提的一点是，各种epoch之间的联系后续会单独开辟一章来解释，这一节还是会略过，这一节主要还是focus在failover状态机上。

```
/* src/sentinel.c */
89 /* Failover machine different states. */
90 #define SENTINEL_FAILOVER_STATE_NONE 0  /* No failover in progress. */
91 #define SENTINEL_FAILOVER_STATE_WAIT_START 1  /* Wait for failover_start_time*/
92 #define SENTINEL_FAILOVER_STATE_SELECT_SLAVE 2 /* Select slave to promote */
93 #define SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE 3 /* Slave -> Master */
94 #define SENTINEL_FAILOVER_STATE_WAIT_PROMOTION 4 /* Wait slave to change role */
95 #define SENTINEL_FAILOVER_STATE_RECONF_SLAVES 5 /* SLAVEOF newmaster */
96 #define SENTINEL_FAILOVER_STATE_UPDATE_CONFIG 6 /* Monitor promoted slave. */

3871 void sentinelFailoverStateMachine(sentinelRedisInstance *ri) {
3872     redisAssert(ri->flags & SRI_MASTER);
3873
3874     if (!(ri->flags & SRI_FAILOVER_IN_PROGRESS)) return;
3875
3876     switch(ri->failover_state) {
3877         case SENTINEL_FAILOVER_STATE_WAIT_START:
3878             sentinelFailoverWaitStart(ri);
3879             break;
3880         case SENTINEL_FAILOVER_STATE_SELECT_SLAVE:
3881             sentinelFailoverSelectSlave(ri);
3882             break;
3883         case SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE:
3884             sentinelFailoverSendSlaveOfNoOne(ri);
3885             break;
3886         case SENTINEL_FAILOVER_STATE_WAIT_PROMOTION:
3887             sentinelFailoverWaitPromotion(ri);
3888             break;
3889         case SENTINEL_FAILOVER_STATE_RECONF_SLAVES:
3890             sentinelFailoverReconfNextSlave(ri);
3891             break;
3892     }
3893 }
```
可以看到sentinelFailoverStateMachine这个状态机的入口是作用于master role的sentinelRedisInstance上的。如果该master role的sentinelRedisInstance没有被置为SRI_FAILOVER_IN_PROGRESS，则该函数也是直接返回。

- **SENTINEL_FAILOVER_STATE_NONE -> SENTINEL_FAILOVER_STATE_WAIT_START**

本阶段是正式进入failover流程的pre阶段。
SRI_FAILOVER_IN_PROGRESS是之前讲到过的sentinelStartFailover那个函数标记的，并且sentinelStartFailover将ri->failover_state由SENTINEL_FAILOVER_STATE_NONE提升为SENTINEL_FAILOVER_STATE_WAIT_START状态.并且这个过程中，除了对failover的master sentinelRedisInstance做出一些比较之外，还不断通过sentinelAskMasterStateToOtherSentinels询问其他的sentinel,寻求投票信息,但是对投票回复信息的汇总不在本阶段，而是在failover正式流程的第一个阶段，即我们即将介绍的下一个阶段。**值得注意的是，之前也提到过，sentinelAskMasterStateToOtherSentinels是一个常态化的作用于master role的sentinelRedisInstance的命令，不是仅限于failover in progress。在failover in progress的区别是ask最后一个参数是该sentinel的runid。**

有关SRI_FAILOVER_IN_PROGRESS以及SENTINEL_FAILOVER_STATE_NONE的变更如下,

```
/* src/sentinel.c */
3459 /* Setup the master state to start a failover. */
3460 void sentinelStartFailover(sentinelRedisInstance *master) {
3461     redisAssert(master->flags & SRI_MASTER);
3462
3463     master->failover_state = SENTINEL_FAILOVER_STATE_WAIT_START;
3464     master->flags |= SRI_FAILOVER_IN_PROGRESS;

3900 void sentinelAbortFailover(sentinelRedisInstance *ri) {
3901     redisAssert(ri->flags & SRI_FAILOVER_IN_PROGRESS);
3902     redisAssert(ri->failover_state <= SENTINEL_FAILOVER_STATE_WAIT_PROMOTION);
3903
3904     ri->flags &= ~(SRI_FAILOVER_IN_PROGRESS|SRI_FORCE_FAILOVER);
3905     ri->failover_state = SENTINEL_FAILOVER_STATE_NONE;

1164 #define SENTINEL_RESET_NO_SENTINELS (1<<0)
1165 void sentinelResetMaster(sentinelRedisInstance *ri, int flags) {
1175     ri->flags &= SRI_MASTER|SRI_DISCONNECTED;
1180     ri->failover_state = SENTINEL_FAILOVER_STATE_NONE;

896 sentinelRedisInstance *createSentinelRedisInstance(char *name, int flags, char *hostname, int port, int quorum, sentinelRedisInstance *master) {
933     ri->flags = flags | SRI_DISCONNECTED;
977     ri->failover_state = SENTINEL_FAILOVER_STATE_NONE;
```

- **SENTINEL_FAILOVER_STATE_WAIT_START -> SENTINEL_FAILOVER_STATE_SELECT_SLAVE**

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
3642     /* If I'm not the leader, and it is not a forced failover via
3643      * SENTINEL FAILOVER, then I can't continue with the failover. */
3644     if (!isleader && !(ri->flags & SRI_FORCE_FAILOVER)) {
3658         return;
3659     }
3660     sentinelEvent(REDIS_WARNING,"+elected-leader",ri,"%@ %llu",
3661         (unsigned long long) ri->failover_epoch);
3662
3663     ri->failover_state = SENTINEL_FAILOVER_STATE_SELECT_SLAVE;
3664     ri->failover_state_change_time = mstime();
3665     sentinelEvent(REDIS_WARNING,"+failover-state-select-slave",ri,"%@");
3666 }
```

可以看到sentinelGetLeader这个函数就是用来统计当前发起ri->failover_epoch版本的failover的投票回复信息的。如果统计出来，截止当前，该sentinel并没有获得选举，没有成为leader角色。而又不是SRI_FORCE_FAILOVER状态的话，则直接返回。值得注意的是SRI_FORCE_FAILOVER的作用就仅仅在此。关于sentinelGetLeader的细节后续会详细介绍。

如果选举成功，则ri->failover_state由SENTINEL_FAILOVER_STATE_WAIT_START提升为SENTINEL_FAILOVER_STATE_SELECT_SLAVE.这一步正式开始进入由此sentinel主导接下来的failover过程。

- **SENTINEL_FAILOVER_STATE_SELECT_SLAVE -> SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE**

```
/* src/sentinel.c */
3668 void sentinelFailoverSelectSlave(sentinelRedisInstance *ri) {
3669     sentinelRedisInstance *slave = sentinelSelectSlave(ri);
3670
3671     /* We don't handle the timeout in this state as the function aborts
3672      * the failover or go forward in the next state. */
3673     if (slave == NULL) {
3674         sentinelEvent(REDIS_WARNING,"-failover-abort-no-good-slave",ri,"%@ %llu",
3675             (unsigned long long) ri->failover_epoch);
3676         sentinelAbortFailover(ri);
3677     } else {
3678         sentinelEvent(REDIS_WARNING,"+selected-slave",slave,"%@");
3679         slave->flags |= SRI_PROMOTED;
3680         ri->promoted_slave = slave;
3681         ri->failover_state = SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE;
3682         ri->failover_state_change_time = mstime();
3683         sentinelEvent(REDIS_NOTICE,"+failover-state-send-slaveof-noone",
3684             slave, "%@");
3685     }
3686 }
```

可以看到，这个阶段的重点是sentinelSelectSlave，如果这个func返回NULL，则表示select slave失败，则直接会终止failover，关于这个failover的失败以及sentinelAbortFailover后续会详细解释.如果func成功返回了一个slave sentinelRedisInstance,则表示这个slave sentinelRedisInstance所指向的redis instance即将被提升为master。所以该slave sentinelRedisInstance的flags被置为SRI_PROMOTED，并且把这个结果也记录在failover的master sentinelRedisInstance的promoted_slave属性中。并且将ri->failover_state由SENTINEL_FAILOVER_STATE_SELECT_SLAVE提升为SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE状态。

详细介绍一下sentinelSelectSlave,

```
/* src/sentinel.c */
3589 sentinelRedisInstance *sentinelSelectSlave(sentinelRedisInstance *master) {
3590     sentinelRedisInstance **instance =
3591         zmalloc(sizeof(instance[0])*dictSize(master->slaves));
3592     sentinelRedisInstance *selected = NULL;
3593     int instances = 0;
3594     dictIterator *di;
3595     dictEntry *de;
3596     mstime_t max_master_down_time = 0;
3597
3598     if (master->flags & SRI_S_DOWN)                                                                                                                                                                    3599         max_master_down_time += mstime() - master->s_down_since_time;
3600     max_master_down_time += master->down_after_period * 10;
3601
3602     di = dictGetIterator(master->slaves);
3603     while((de = dictNext(di)) != NULL) {
3604         sentinelRedisInstance *slave = dictGetVal(de);
3605         mstime_t info_validity_time;
3606
3607         if (slave->flags & (SRI_S_DOWN|SRI_O_DOWN|SRI_DISCONNECTED)) continue;
3608         if (mstime() - slave->last_avail_time > SENTINEL_PING_PERIOD*5) continue;
3609         if (slave->slave_priority == 0) continue;
3610
3611         /* If the master is in SDOWN state we get INFO for slaves every second.                                                                                                                        3612          * Otherwise we get it with the usual period so we need to account for
3613          * a larger delay. */
3614         if (master->flags & SRI_S_DOWN)
3615             info_validity_time = SENTINEL_PING_PERIOD*5;
3616         else
3617             info_validity_time = SENTINEL_INFO_PERIOD*3;
3618         if (mstime() - slave->info_refresh > info_validity_time) continue;                                                                                                                             3619         if (slave->master_link_down_time > max_master_down_time) continue;
3620         instance[instances++] = slave;                                                                                                                                                                 3621     }
3622     dictReleaseIterator(di);
3623     if (instances) {
3624         qsort(instance,instances,sizeof(sentinelRedisInstance*),                                                                                                                                       3625             compareSlavesForPromotion);                                                                                                                                                                3626         selected = instance[0];
3627     }
3628     zfree(instance);
3629     return selected;
3630 }
```
可以看到这个函数的目的就是从优挑一个挂载在failover的master sentinelRedisInstance下的slave sentinelRedisInstance作为被选中的slave返回。

以下几种情况的slave sentinelRedisInstance会被pass掉。

- 该slave sentinelRedisInstance处于SRI_S_DOWN|SRI_O_DOWN|SRI_DISCONNECTED状态

- 该slave sentinelRedisInstance的last_avail_time距今超过5倍SENTINEL_PING_PERIOD时间，last_avail_tim的意义本来就是记录上一次对ping cmd的acceptable reply的时间。

- 该slave sentinelRedisInstance记录的指向的slave的instance的配置项slave_priority为0.

- 该slave sentinelRedisInstance的info_refresh距今超过5倍SENTINEL_PING_PERIOD（master sentinelRedisInstance处于sdown状态）否则是3倍SENTINEL_INFO_PERIOD。这个区别是因为在sdown的情况下，info的频率就是1000ms和默认的SENTINEL_PING_PERIOD相同。这种情况下我们准许的delay要比正常情况小一些。
 
	```
	/* src/sentinel.c */
	2344 void sentinelSendPeriodicCommands(sentinelRedisInstance *ri) {
	2361     /* If this is a slave of a master in O_DOWN condition we start sending
	2362      * it INFO every second, instead of the usual SENTINEL_INFO_PERIOD
	2363      * period. In this state we want to closely monitor slaves in case they                                                                                                                            	2364      * are turned into masters by another Sentinel, or by the sysadmin. */
	2365     if ((ri->flags & SRI_SLAVE) &&
	2366         (ri->master->flags & (SRI_O_DOWN|SRI_FAILOVER_IN_PROGRESS))) {
	2367         info_period = 1000;
	2368     } else {
	2369         info_period = SENTINEL_INFO_PERIOD;
	2370     }
	```
- 该slave反馈的master_link_down_time大于max_master_down_time。

	关于max_master_down_time这个值，如果failover的master sentinelRedisInstance处于SRI_S_DOWN，则给max_master_down_time加上mstime() - master->s_down_since_time，就是master sentinelRedisInstance记录的该master instance进入sdown状态已经持续了多长时间。再加上10倍master->down_after_period(默认为30s)，默认情况下，就是300秒，5分钟。max_master_down_time加上10倍master->down_after_period就是给redis instance判断master_link_down_time等信息预留了一部分时间，如果超过这个时间，则认为该slave的master_link_down_time太久。涉及到的master->s_down_since_time和slave->master_link_down_time这里解释一下，两者其实没有直接关联。
	
	- master->s_down_since_time是由于该master sentinelRedisInstance本身收集的一些状态触发的。之前已经介绍过了。
	
	- slave->master_link_down_time则是有slave sentinelRedisInstance记录的远程instance的info回复信息中的信息。

经过以上条件筛选剩下来的slave丢入qsort按compareSlavesForPromotion里定义的优先级进行快排，取排序后的第一个。

```
/* src/sentinel.c */
3558 /* Helper for sentinelSelectSlave(). This is used by qsort() in order to
3559  * sort suitable slaves in a "better first" order, to take the first of
3560  * the list. */
3561 int compareSlavesForPromotion(const void *a, const void *b) {
3562     sentinelRedisInstance **sa = (sentinelRedisInstance **)a,
3563                           **sb = (sentinelRedisInstance **)b;
3564     char *sa_runid, *sb_runid;
3565
3566     if ((*sa)->slave_priority != (*sb)->slave_priority)
3567         return (*sa)->slave_priority - (*sb)->slave_priority;
3568
3569     /* If priority is the same, select the slave with greater replication
3570      * offset (processed more data frmo the master). */
3571     if ((*sa)->slave_repl_offset > (*sb)->slave_repl_offset) {
3572         return -1; /* a < b */
3573     } else if ((*sa)->slave_repl_offset < (*sb)->slave_repl_offset) {
3574         return 1; /* b > a */
3575     }
3576
3577     /* If the replication offset is the same select the slave with that has
3578      * the lexicographically smaller runid. Note that we try to handle runid
3579      * == NULL as there are old Redis versions that don't publish runid in
3580      * INFO. A NULL runid is considered bigger than any other runid. */
3581     sa_runid = (*sa)->runid;
3582     sb_runid = (*sb)->runid;
3583     if (sa_runid == NULL && sb_runid == NULL) return 0;
3584     else if (sa_runid == NULL) return 1;  /* a > b */
3585     else if (sb_runid == NULL) return -1; /* a < b */
3586     return strcasecmp(sa_runid, sb_runid);
3587 }
```

分以下几个层次

- 先比较slave_priority，越小越好。

- 再比较slave_repl_offset，越小越好。

- runid字母序strcasecmp，越小越好。

可以看到经历整个过程选出来的slave，并不能保证slave instance本身的状态，以及slave对数据的同步状态。

- **SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE -> SENTINEL_FAILOVER_STATE_WAIT_PROMOTION**

```
/* src/sentinel.c */
3688 void sentinelFailoverSendSlaveOfNoOne(sentinelRedisInstance *ri) {
3689     int retval;
3690
3691     /* We can't send the command to the promoted slave if it is now
3692      * disconnected. Retry again and again with this state until the timeout
3693      * is reached, then abort the failover. */
3694     if (ri->promoted_slave->flags & SRI_DISCONNECTED) {
3701         return;
3702     }
3703
3704     /* Send SLAVEOF NO ONE command to turn the slave into a master.
3705      * We actually register a generic callback for this command as we don't
3706      * really care about the reply. We check if it worked indirectly observing
3707      * if INFO returns a different role (master instead of slave). */
3708     retval = sentinelSendSlaveOf(ri->promoted_slave,NULL,0);
3709     if (retval != REDIS_OK) return;
3710     sentinelEvent(REDIS_NOTICE, "+failover-state-wait-promotion",
3711         ri->promoted_slave,"%@");
3712     ri->failover_state = SENTINEL_FAILOVER_STATE_WAIT_PROMOTION;
3713     ri->failover_state_change_time = mstime();                                                                                                                                                         3714 }
```
这个阶段是整个failover过程中最关键的阶段,如果该ri->promoted_slave的flags表示处于SRI_DISCONNECTED，则直接退出，或者sentinelSendSlaveOf的返回值不是ok的话，也会直接退出，这两种情况下的直接退出会导致后续再次进入此函数逻辑进行重试。如果没有上面两种提前终止，则会将ri->failover_state由SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE提升为SENTINEL_FAILOVER_STATE_WAIT_PROMOTION状态。可以看到，对于sentinelSendSlaveOf产生的影响的评估是在下个阶段进行处理。

重点提一下sentinelSendSlaveOf(ri->promoted_slave,NULL,0)这句话会产生的影响是让该ri->promoted_slave sentinelRedisInstance所指向的远程redis slave instance断掉现有master的sync链接，以备马上就会有其他的slave instance以他为master进行sync。至于该promoted_slave此时作为master是否合格，数据同步是否没有跟上而产生数据丢失，都是快刀斩乱麻一刀切，不归这个动作管, sentinel在之前select slave状态对上面的担心做了一定的评估。不过正如之前提到的并没有完全保证不丢数据。

```
/* src/sentinel.c */
3393 /* Send SLAVEOF to the specified instance, always followed by a
3394  * CONFIG REWRITE command in order to store the new configuration on disk
3395  * when possible (that is, if the Redis instance is recent enough to support
3396  * config rewriting, and if the server was started with a configuration file).
3397  *
3398  * If Host is NULL the function sends "SLAVEOF NO ONE".
3399  *
3400  * The command returns REDIS_OK if the SLAVEOF command was accepted for
3401  * (later) delivery otherwise REDIS_ERR. The command replies are just
3402  * discarded. */
3403 int sentinelSendSlaveOf(sentinelRedisInstance *ri, char *host, int port) {
3404     char portstr[32];
3405     int retval;
3406
3407     ll2string(portstr,sizeof(portstr),port);
3408
3409     /* If host is NULL we send SLAVEOF NO ONE that will turn the instance
3410      * into a master. */
3411     if (host == NULL) {
3412         host = "NO";
3413         memcpy(portstr,"ONE",4);
3414     }
3415
3416     /* In order to send SLAVEOF in a safe way, we send a transaction performing
3417      * the following tasks:
3418      * 1) Reconfigure the instance according to the specified host/port params.
3419      * 2) Rewrite the configuraiton.
3420      * 3) Disconnect all clients (but this one sending the commnad) in order
3421      *    to trigger the ask-master-on-reconnection protocol for connected
3422      *    clients.
3423      *
3424      * Note that we don't check the replies returned by commands, since we
3425      * will observe instead the effects in the next INFO output. */
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
3439     ri->pending_commands++;
3440
3441     /* CLIENT KILL TYPE <type> is only supported starting from Redis 2.8.12,
3442      * however sending it to an instance not understanding this command is not
3443      * an issue because CLIENT is variadic command, so Redis will not
3444      * recognized as a syntax error, and the transaction will not fail (but
3445      * only the unsupported command will fail). */
3446     retval = redisAsyncCommand(ri->cc,
3447         sentinelDiscardReplyCallback, NULL, "CLIENT KILL TYPE normal");
3448     if (retval == REDIS_ERR) return retval;
3449     ri->pending_commands++;
3450
3451     retval = redisAsyncCommand(ri->cc,
3452         sentinelDiscardReplyCallback, NULL, "EXEC");
3453     if (retval == REDIS_ERR) return retval;
3454     ri->pending_commands++;
3455
3456     return REDIS_OK;
3457 }
```
这个函数的返回值很有意思，The command returns REDIS_OK if the SLAVEOF command was accepted for (later) delivery otherwise REDIS_ERR.其实是任意其中任意的cmd失败，都会返回REDIS_ERR。

send SLAVEOF是一个transaction，暂时不谈redis的transaction机制，暂时我也不了解。包含以下命令。

- SLAVEOF命令，如果host是NO,port是ONE，则表示不slave of任何master，自己是master的意思。

- CONFIG REWRITE的意思是该instance响应执行了slave of之后，马上将slave变更写入config文件。

- CLIENT KILL TYPE normal是指从redis instance端杀掉所有normal type的连接，除当前发送命令的connection外。 官方文档对此的解释是，

	> CLIENT KILL and Redis Sentinel Recent versions of Redis Sentinel (Redis 2.8.12 or greater) use CLIENT KILL in order to kill clients when an instance is reconfigured, in order to force clients to perform the handshake with one Sentinel again and update its configuration
	
	> CLIENT KILL TYPE type, where type is one of normal, slave, pubsub. This closes the connections of all the clients in the specified class. Note that clients blocked into the MONITOR command are considered to belong to the normal class.
	
上面引用中提到的client type跟sentinelSetClientName时指定的name没有直接关系,虽然name的格式是sentinel-<first_8_chars_of_runid>-<connection_type> % ("cmd" or "pubsub")。sentinelSetClientName是为了方便使用者的角度来grep的。

```
/* src/networking.c */
1544 /* Get the class of a client, used in order to enforce limits to different
1545  * classes of clients.
1546  *
1547  * The function will return one of the following:
1548  * REDIS_CLIENT_TYPE_NORMAL -> Normal client
1549  * REDIS_CLIENT_TYPE_SLAVE  -> Slave or client executing MONITOR command
1550  * REDIS_CLIENT_TYPE_PUBSUB -> Client subscribed to Pub/Sub channels
1551  */
1552 int getClientType(redisClient *c) {
1553     if ((c->flags & REDIS_SLAVE) && !(c->flags & REDIS_MONITOR))
1554         return REDIS_CLIENT_TYPE_SLAVE;
1555     if (c->flags & REDIS_PUBSUB)
1556         return REDIS_CLIENT_TYPE_PUBSUB;
1557     return REDIS_CLIENT_TYPE_NORMAL;
1558 }

1560 int getClientTypeByName(char *name) {
1561     if (!strcasecmp(name,"normal")) return REDIS_CLIENT_TYPE_NORMAL;
1562     else if (!strcasecmp(name,"slave")) return REDIS_CLIENT_TYPE_SLAVE;
1563     else if (!strcasecmp(name,"pubsub")) return REDIS_CLIENT_TYPE_PUBSUB;
1564     else return -1;
1565 }
```

- **SENTINEL_FAILOVER_STATE_WAIT_PROMOTION -> SENTINEL_FAILOVER_STATE_RECONF_SLAVES**

```
/* src/sentinel.c */
3716 /* We actually wait for promotion indirectly checking with INFO when the
3717  * slave turns into a master. */
3718 void sentinelFailoverWaitPromotion(sentinelRedisInstance *ri) {
3719     /* Just handle the timeout. Switching to the next state is handled
3720      * by the function parsing the INFO command of the promoted slave. */
3721     if (mstime() - ri->failover_state_change_time > ri->failover_timeout) {
3725         sentinelAbortFailover(ri);
3726     }
3727 }
```

可以看出来，这个sentinelFailoverWaitPromotion实际没有关于failover的直接操作，包括到下一个状态的提升也没有。只是用来判断这个状态如果持续太长时间了判定超时。实际的逻辑正如indirectly checking with INFO when the slave turns into a master.这句comment提到的一样，在sentinelRefreshInstanceInfo这个info reply callback中。

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
```

其中

```
/* src/sentinel.c */
1948         if ((ri->flags & SRI_PROMOTED) &&
1949             (ri->master->flags & SRI_FAILOVER_IN_PROGRESS) &&
1950             (ri->master->failover_state ==
1951                 SENTINEL_FAILOVER_STATE_WAIT_PROMOTION))
```

讨论过上面这个if判读为假的逻辑，现在来讨论为真的逻辑，也就是本节需要的状态提升逻辑。这个if的含义很明显。
这里我们可以明确看到ri->master->failover_state由SENTINEL_FAILOVER_STATE_WAIT_PROMOTION提升为SENTINEL_FAILOVER_STATE_RECONF_SLAVES。相关的epoch逻辑以及sentinelForceHelloUpdateForMaster逻辑后续会详细解释。

干脆在这里提前讲一下sentinelAbortFailover，这个函数用于非自然结束一个failover,并重置一些failover的master sentinelRedisInstance的状态。这个函数能够继续走下去的前提是failover_state没有由SENTINEL_FAILOVER_STATE_WAIT_PROMOTION提升为SENTINEL_FAILOVER_STATE_RECONF_SLAVES。ri->failover_state <= SENTINEL_FAILOVER_STATE_WAIT_PROMOTION。**这只是一个best effort的动作，终止了failover，但是slave of no one的命令可能会在终止后产生影响(但是终止时还没有info到该影响)，将promoted_slave instance提升为master instance提升，但是不是破坏性的，此时就会产生该slave被转换为master，但是不被任何人承认的问题。之前提到过的sentinelRefreshInstanceInfo中的+convert-to-slave逻辑会修复这个问题。所以从此看来，更需要监控+convert-to-slave 这个message并尽量避免。为什么不是破坏性的，首先是因为这个mismatch并不会被广播出去，再加上该redis instance是slave角色，所以这个角色更改在外部是无法感知的，唯一可能的影响就是此时恰好有client连接这个instance并写入了data，这个data在这后肯定会被丢掉。最后该问题会在大概4倍SENTINEL_PUBLISH_PERIOD被自动修复。**

```
/* src/sentinel.c */
3895 /* Abort a failover in progress:
3896  *
3897  * This function can only be called before the promoted slave acknowledged
3898  * the slave -> master switch. Otherwise the failover can't be aborted and
3899  * will reach its end (possibly by timeout). */
3900 void sentinelAbortFailover(sentinelRedisInstance *ri) {
3901     redisAssert(ri->flags & SRI_FAILOVER_IN_PROGRESS);
3902     redisAssert(ri->failover_state <= SENTINEL_FAILOVER_STATE_WAIT_PROMOTION);
3903
3904     ri->flags &= ~(SRI_FAILOVER_IN_PROGRESS|SRI_FORCE_FAILOVER);
3905     ri->failover_state = SENTINEL_FAILOVER_STATE_NONE;
3906     ri->failover_state_change_time = mstime();
3907     if (ri->promoted_slave) {
3908         ri->promoted_slave->flags &= ~SRI_PROMOTED;
3909         ri->promoted_slave = NULL;
3910     }
3911 }
```

**SENTINEL_FAILOVER_STATE_RECONF_SLAVES -> SENTINEL_FAILOVER_STATE_UPDATE_CONFIG**

```
/* src/sentinel.c */
3795 /* Send SLAVE OF <new master address> to all the remaining slaves that
3796  * still don't appear to have the configuration updated. */
3797 void sentinelFailoverReconfNextSlave(sentinelRedisInstance *master) {
3798     dictIterator *di;
3799     dictEntry *de;
3800     int in_progress = 0;
3801
3802     di = dictGetIterator(master->slaves);
3803     while((de = dictNext(di)) != NULL) {
3804         sentinelRedisInstance *slave = dictGetVal(de);
3805
3806         if (slave->flags & (SRI_RECONF_SENT|SRI_RECONF_INPROG))
3807             in_progress++;
3808     }
3809     dictReleaseIterator(di);
3810
3811     di = dictGetIterator(master->slaves);
3812     while(in_progress < master->parallel_syncs &&
3813           (de = dictNext(di)) != NULL)
3814     {
3815         sentinelRedisInstance *slave = dictGetVal(de);
3816         int retval;
3817
3818         /* Skip the promoted slave, and already configured slaves. */
3819         if (slave->flags & (SRI_PROMOTED|SRI_RECONF_DONE)) continue;
3833
3834         /* Nothing to do for instances that are disconnected or already
3835          * in RECONF_SENT state. */
3836         if (slave->flags & (SRI_DISCONNECTED|SRI_RECONF_SENT|SRI_RECONF_INPROG))
3837             continue;
```

这个函数是在promoted_slave提升为master后，将向failover的old master sentinelRedisInstance挂载下的所有slave sentinelRedisInstance(除promoted_slave new master之外)所指向的redis slave instance发送sentinelSendSlaveOf到new master，此时还记录在old master->promoted_slave中。

以下几种情况会跳过或者暂时跳过，

- 此处会统计在SRI_RECONF_SENT|SRI_RECONF_INPROG状态的slave sentinelRedisInstance的个数。如果达到上限master->parallel_syncs，则慢慢来，主要是控制同时与new master sync的数量。master->parallel_syncs的作用就在此，并且之前也提过。

- 如果该slave sentinelRedisInstance是SRI_PROMOTED状态，即promoted_slave，the new master。此处就是SRI_PROMOTED的又一作用。

- 如果该slave sentinelRedisInstance已经是SRI_RECONF_DONE状态，则表示不仅已经sentinelSendSlaveOf过了，并且按照cmd的意思slave instance和new master instance之间已经config好了, master_link已经建立好了，master_link_status已经是ok的了。之前讲SRI_RECONF_xx系列时也提到过，master_link_status还是后续再讲。

- 如果该slave sentinelRedisInstance处于SRI_DISCONNECTED|SRI_RECONF_SENT|SRI_RECONF_INPROG这三种状态，都是显而易见的策略。

sentinelFailoverReconfNextSlave剩余部分，

```
/* src/sentinel.c */
3797 void sentinelFailoverReconfNextSlave(sentinelRedisInstance *master) {
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
3849     }
3850     dictReleaseIterator(di);
3851
3852     /* Check if all the slaves are reconfigured and handle timeout. */
3853     sentinelFailoverDetectEnd(master);
```
就是执行sentinelSendSlaveOf，并且将slave sentinelRedisInstance置为SRI_RECONF_SENT SRI_RECONF_xx系列的初始状态,并且初始化了slave sentinelRedisInstance的slave_reconf_sent_time属性为当前时间，主要是用于判断超时的时候会用到，后续会解释，至此也就是SRI_RECONF_xx系列的最后没有提到过的在sentinelFailoverReconfNextSlave中的作用。

最后讲讲sentinelFailoverDetectEnd这个函数，最重要的failover state提升的逻辑还在里面呢。

```
/* src/sentinel.c */
3729 void sentinelFailoverDetectEnd(sentinelRedisInstance *master) {
3730     int not_reconfigured = 0, timeout = 0;
3731     dictIterator *di;
3732     dictEntry *de;
3733     mstime_t elapsed = mstime() - master->failover_state_change_time;
3734
3735     /* We can't consider failover finished if the promoted slave is
3736      * not reachable. */
3737     if (master->promoted_slave == NULL ||
3738         master->promoted_slave->flags & SRI_S_DOWN) return;
3739
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
3759
3760     if (not_reconfigured == 0) {
3761         sentinelEvent(REDIS_WARNING,"+failover-end",master,"%@ %llu",
3762             (unsigned long long) master->failover_epoch);
3763
3764         master->failover_state = SENTINEL_FAILOVER_STATE_UPDATE_CONFIG;
3765         master->failover_state_change_time = mstime();
3766     }
```

如果master->promoted_slave->flags表示promoted_slave sentinelRedisInstance处于SRI_S_DOWN状态是not reachable，则此处直接返回供后续重试。

然后统计所有reachable的slave sentinelRedisInstance中还没有被reconfg的数量，注意这里如果slave sentinelRedisInstance处于SRI_S_DOWN状态也会略过，不计入not_reconfigured。

如果上面的策略统计出来not_reconfigured为0，则将master->failover_state由SENTINEL_FAILOVER_STATE_RECONF_SLAVES提升为SENTINEL_FAILOVER_STATE_UPDATE_CONFIG状态，并且输出+failover-end的消息，至此failover的主要使命就完成了，不过还有一些收尾操作，后续马上会讲。

至此sentinelFailoverStateMachine就讲完了，并且sentinelHandleRedisInstance就讲完了.

### **after failover end success**
----------------------------------

SENTINEL_FAILOVER_STATE_UPDATE_CONFIG之后，就是failover主要流程结束后的一些config操作,在本sentinel instance的old master sentinelRedisInstance stuct以及promoted_slave sentinelRedisInstance stuct都还未被重新config。并且此failover产生的这些变动也还未被广播出去，有关广播出去这部分内容这节估计会带过.

这部分操作是在sentinelHandleDictOfRedisInstances进行的。

```
/* src/sentinel.c */
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
3969             if (ri->failover_state == SENTINEL_FAILOVER_STATE_UPDATE_CONFIG) {
3970                 switch_to_promoted = ri;
3971             }
3972         }
3973     }
3974     if (switch_to_promoted)
3975         sentinelFailoverSwitchToPromotedSlave(switch_to_promoted);
3976     dictReleaseIterator(di);
3977 }
```

sentinelHandleDictOfRedisInstances的递归调用的，并且是在sentinelTimer中直接调用的，参数是sentinel.masters。

如果某一个master sentinelRedisInstance的failover_state处于SENTINEL_FAILOVER_STATE_UPDATE_CONFIG状态，则在该master sentinelRedisInstance执行sentinelFailoverSwitchToPromotedSlave逻辑，可以看到一次sentinelHandleDictOfRedisInstances调用，只会有一个failover_state处于SENTINEL_FAILOVER_STATE_UPDATE_CONFIG的master sentinelRedisInstance会被处理。

```
/* src/sentinel.c */
3856 /* This function is called when the slave is in
3857  * SENTINEL_FAILOVER_STATE_UPDATE_CONFIG state. In this state we need
3858  * to remove it from the master table and add the promoted slave instead. */
3859 void sentinelFailoverSwitchToPromotedSlave(sentinelRedisInstance *master) {
3860     sentinelRedisInstance *ref = master->promoted_slave ?
3861                                  master->promoted_slave : master;
3862
3863     sentinelEvent(REDIS_WARNING,"+switch-master",master,"%s %s %d %s %d %llu",
3864         master->name, master->addr->ip, master->addr->port,
3865         ref->addr->ip, ref->addr->port,
3866         (unsigned long long) master->failover_epoch);
3867
3868     sentinelResetMasterAndChangeAddress(master,ref->addr->ip,ref->addr->port);
3869 }
```
此函数输出了我们最关心的+switch-master message，后续会详细解释。**此处ref = master->promoted_slave ?  master->promoted_slave : master;的逻辑，我个人认为是多余的master->promoted_slave肯定为真，不知作者的考虑是什么,还是说是一个无关紧要的mistake**。接下来详细解释一下sentinelResetMasterAndChangeAddress函数,

```
/* src/sentinel.c */
1219 /* Reset the specified master with sentinelResetMaster(), and also change
1220  * the ip:port address, but take the name of the instance unmodified.
1221  *
1222  * This is used to handle the +switch-master event.
1223  *
1224  * The function returns REDIS_ERR if the address can't be resolved for some
1225  * reason. Otherwise REDIS_OK is returned.  */
1226 int sentinelResetMasterAndChangeAddress(sentinelRedisInstance *master, char *ip, int port) 
1233     newaddr = createSentinelAddr(ip,port);
1234     if (newaddr == NULL) return REDIS_ERR;
1235
1236     /* Make a list of slaves to add back after the reset.
1237      * Don't include the one having the address we are switching to. */
1238     di = dictGetIterator(master->slaves);
1239     while((de = dictNext(di)) != NULL) {
1240         sentinelRedisInstance *slave = dictGetVal(de);
1241
1242         if (sentinelAddrIsEqual(slave->addr,newaddr)) continue;
1243         slaves = zrealloc(slaves,sizeof(sentinelAddr*)*(numslaves+1));
1244         slaves[numslaves++] = createSentinelAddr(slave->addr->ip,
1245                                                  slave->addr->port);
1246     }
1247     dictReleaseIterator(di);
1248
1249     /* If we are switching to a different address, include the old address
1250      * as a slave as well, so that we'll be able to sense / reconfigure
1251      * the old master. */
1252     if (!sentinelAddrIsEqual(newaddr,master->addr)) {
1253         slaves = zrealloc(slaves,sizeof(sentinelAddr*)*(numslaves+1));
1254         slaves[numslaves++] = createSentinelAddr(master->addr->ip,
1255                                                  master->addr->port);
1256     }
1257
1258     /* Reset and switch address. */
1259     sentinelResetMaster(master,SENTINEL_RESET_NO_SENTINELS);
1260     oldaddr = master->addr;
1261     master->addr = newaddr;
1262     master->o_down_since_time = 0;
1263     master->s_down_since_time = 0;
1264
1265     /* Add slaves back. */
1266     for (j = 0; j < numslaves; j++) {
1267         sentinelRedisInstance *slave;
1268
1269         slave = createSentinelRedisInstance(NULL,SRI_SLAVE,slaves[j]->ip,
1270                     slaves[j]->port, master->quorum, master);
1271         releaseSentinelAddr(slaves[j]);
1272         if (slave) {
1273             sentinelEvent(REDIS_NOTICE,"+slave",slave,"%@");
1274             sentinelFlushConfig();
1275         }
1276     }
1277     zfree(slaves);
```

除去sentinelResetMaster(master,SENTINEL_RESET_NO_SENTINELS);的逻辑暂时不说，这个func主要做了以下几件事。

- 先将master sentinelRedisInstance下挂载的所有slave sentinelRedisInstance backup到local slaves中，除去switch to的promoted_slave外。

- 将现有old master sentinelRedisInstance也加入到local slaves中，如果old master的addr同switch to 的addr不同的话。包括sentinelAddrIsEqual在内的这一系列sentinelAddr函数很简单，我就任性一把，不讲了.

- backup到slaves做好之后，调用sentinelResetMaster该master sentinelRedisInstance，并且将该master sentinelRedisInstance的addr替换为新的switch to的addr。并且重置master->o_down_since_time，master->s_down_since_time两个属性。至此之前的master sentinelRedisInstance已从old master的配置切换到switch to 的new master配置。**值得注意的是sentinelResetMaster中的一个重要逻辑是将ri->failover_state重置为 SENTINEL_FAILOVER_STATE_NONE,表示全部退出failover的流程，恢复到正常状态。**

- iter刚才backup的local slaves，全部重新创建slave sentinelRedisInstance并挂载在master sentinelRedisInstance下。至此此sentinel下的该master sentinelRedisInstance以及其slave sentinelRedisInstance的状态都重置了一遍，注意SENTINEL_RESET_NO_SENTINELS表示并没有更新任何该master sentinelRedisInstance下挂载的sentinel sentinelRedisInstance。

至此，sentinelHandleDictOfRedisInstances也就介绍完了，并且failover状态机的大部分内容已经讲完。

### **failover end fail**
-------------------------

上面的章节分阶段讲了sentinel failover success的唯一路径，但是sentinel failover中失败会有很多路径。还是分阶段讲在每个阶段失败的可能性.

首先提一下failover_state_change_time这个属性，

```
/* src/sentinel.c */
896 sentinelRedisInstance *createSentinelRedisInstance(char *name, int flags, char *hostname, int port, int quorum, sentinelRedisInstance *master) {
977     ri->failover_state = SENTINEL_FAILOVER_STATE_NONE;
978     ri->failover_state_change_time = 0;

1165 void sentinelResetMaster(sentinelRedisInstance *ri, int flags) {
1180     ri->failover_state = SENTINEL_FAILOVER_STATE_NONE;
1181     ri->failover_state_change_time = 0;

1789 /* Process the INFO output from masters. */
1790 void sentinelRefreshInstanceInfo(sentinelRedisInstance *ri, const char *info) {
1959             ri->master->failover_state = SENTINEL_FAILOVER_STATE_RECONF_SLAVES;
1960             ri->master->failover_state_change_time = mstime();

3460 void sentinelStartFailover(sentinelRedisInstance *master) {
3463     master->failover_state = SENTINEL_FAILOVER_STATE_WAIT_START;
3477     master->failover_state_change_time = mstime();

3633 void sentinelFailoverWaitStart(sentinelRedisInstance *ri) {
3663     ri->failover_state = SENTINEL_FAILOVER_STATE_SELECT_SLAVE;
3664     ri->failover_state_change_time = mstime();

3668 void sentinelFailoverSelectSlave(sentinelRedisInstance *ri) {
3681         ri->failover_state = SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE;
3682         ri->failover_state_change_time = mstime();

3688 void sentinelFailoverSendSlaveOfNoOne(sentinelRedisInstance *ri) {
3712     ri->failover_state = SENTINEL_FAILOVER_STATE_WAIT_PROMOTION;
3713     ri->failover_state_change_time = mstime();

3729 void sentinelFailoverDetectEnd(sentinelRedisInstance *master) {
3764         master->failover_state = SENTINEL_FAILOVER_STATE_UPDATE_CONFIG;
3765         master->failover_state_change_time = mstime();

3900 void sentinelAbortFailover(sentinelRedisInstance *ri) {
3905     ri->failover_state = SENTINEL_FAILOVER_STATE_NONE;
3906     ri->failover_state_change_time = mstime();
```

可以看到failover_state_change_time这个属性在任何failover_state变更的地方都会随之变更，除了变更为SENTINEL_FAILOVER_STATE_NONE会将failover_state_change_time置为0之外，都是置为mstime()，这些变更是干嘛的呢，马上会提到。

首先定义所谓failover fail终止提到的是，这些fail逻辑下，会采取一些措施，阻止failover流程稍后继续重试，如sentinelAbortFailover等，而如果是那种直接return的逻辑，后续还是会从该阶段往后重试，这种类型的暂时终止在内。

**SENTINEL_FAILOVER_STATE_NONE -> SENTINEL_FAILOVER_STATE_WAIT_START**

这个阶段会有一些条件阻止failover_state的提升，即正式进入failover流程。但是由于还没进入failover的流程，这一阶段的失败并不能算是failover的failover终止.

**SENTINEL_FAILOVER_STATE_WAIT_START -> SENTINEL_FAILOVER_STATE_SELECT_SLAVE**

```
/* src/sentinel.c */
3632 /* ---------------- Failover state machine implementation ------------------- */
3633 void sentinelFailoverWaitStart(sentinelRedisInstance *ri) {
3634     char *leader;
3635     int isleader;
3636
3637     /* Check if we are the leader for the failover epoch. */
3638     leader = sentinelGetLeader(ri, ri->failover_epoch);
3639     isleader = leader && strcasecmp(leader,server.runid) == 0;
3640     sdsfree(leader);
3641
3642     /* If I'm not the leader, and it is not a forced failover via
3643      * SENTINEL FAILOVER, then I can't continue with the failover. */
3644     if (!isleader && !(ri->flags & SRI_FORCE_FAILOVER)) {
3645         int election_timeout = SENTINEL_ELECTION_TIMEOUT;
3646
3647         /* The election timeout is the MIN between SENTINEL_ELECTION_TIMEOUT
3648          * and the configured failover timeout. */
3649         if (election_timeout > ri->failover_timeout)
3650             election_timeout = ri->failover_timeout;
3651         /* Abort the failover if I'm not the leader after some time. */
3652         if (mstime() - ri->failover_start_time > election_timeout) {
3653             sentinelEvent(REDIS_WARNING,"-failover-abort-not-elected",ri,"%@ %llu",
3654                 (unsigned long long) ri->failover_epoch);
3655
3656             sentinelAbortFailover(ri);
3657         }
3658         return;
3659     }
```

可以看到如果sentinelGetLeader统计出的该次failover_epoch的failover的leader不是当前sentinel。并且也不是SRI_FORCE_FAILOVER这种强制人工指定failover的状态，则给出了一个min(SENTINEL_ELECTION_TIMEOUT, ri->failover_timeout)的election_timeout时间,SENTINEL_ELECTION_TIMEOUT默认是10s， 如果ri->failover_start_time距今已经超过election_timeout时间，则认为这么长时间内，当前sentinel选举失败，要么就是选票被别人拿走了，要么就是大家都没成为大多数等等情况，则算-failover-abort-not-elected，并且sentinelAbortFailover。关于vote以及failover_start_time的细节，后续会详细解释。

**SENTINEL_FAILOVER_STATE_SELECT_SLAVE -> SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE**

```
/* src/sentinel.c */
3668 void sentinelFailoverSelectSlave(sentinelRedisInstance *ri) {
3669     sentinelRedisInstance *slave = sentinelSelectSlave(ri);
3670
3671     /* We don't handle the timeout in this state as the function aborts
3672      * the failover or go forward in the next state. */
3673     if (slave == NULL) {
3674         sentinelEvent(REDIS_WARNING,"-failover-abort-no-good-slave",ri,"%@ %llu",
3675             (unsigned long long) ri->failover_epoch);
3676         sentinelAbortFailover(ri);
```

可以看到如果此处sentinelSelectSlave选不出来合格的slave，则算-failover-abort-no-good-slave，并且sentinelAbortFailover。

**SENTINEL_FAILOVER_STATE_SEND_SLAVEOF_NOONE -> SENTINEL_FAILOVER_STATE_WAIT_PROMOTION**

```
/* src/sentinel.c */
3688 void sentinelFailoverSendSlaveOfNoOne(sentinelRedisInstance *ri) {
3689     int retval;
3690
3691     /* We can't send the command to the promoted slave if it is now
3692      * disconnected. Retry again and again with this state until the timeout
3693      * is reached, then abort the failover. */
3694     if (ri->promoted_slave->flags & SRI_DISCONNECTED) {
3695         if (mstime() - ri->failover_state_change_time > ri->failover_timeout) {
3696             sentinelEvent(REDIS_WARNING,"-failover-abort-slave-timeout",ri,"%@ %llu",
3697                 (unsigned long long) ri->failover_epoch);
3698
3699             sentinelAbortFailover(ri);
3700         }
3701         return;
3702     }
```
可以看到此阶段如果ri->promoted_slave处于SRI_DISCONNECTED，并且距离上一次更新ri->failover_state_change_time已经超过ri->failover_timeout，则算-failover-abort-slave-timeout，并sentinelAbortFailover。
  
**SENTINEL_FAILOVER_STATE_WAIT_PROMOTION -> SENTINEL_FAILOVER_STATE_RECONF_SLAVES**

```
/* src/sentinel.c */
3716 /* We actually wait for promotion indirectly checking with INFO when the
3717  * slave turns into a master. */
3718 void sentinelFailoverWaitPromotion(sentinelRedisInstance *ri) {
3719     /* Just handle the timeout. Switching to the next state is handled
3720      * by the function parsing the INFO command of the promoted slave. */
3721     if (mstime() - ri->failover_state_change_time > ri->failover_timeout) {
3722         sentinelEvent(REDIS_WARNING,"-failover-abort-slave-timeout",ri,"%@ %llu",
3723             (unsigned long long) ri->failover_epoch);
3724
3725         sentinelAbortFailover(ri);
3726     }
3727 }
```
可以看到这里的主要逻辑就是判断进入这个failover_state后，上次ri->failover_state_change_time更新也几乎是在同时，如果距离上次ri->failover_state_change_time更新时间已经超过ri->failover_timeout，则算-failover-abort-slave-timeout，并sentinelAbortFailover。

**SENTINEL_FAILOVER_STATE_RECONF_SLAVES -> SENTINEL_FAILOVER_STATE_UPDATE_CONFIG**

```
/* src/sentinel.c */
3795 /* Send SLAVE OF <new master address> to all the remaining slaves that
3796  * still don't appear to have the configuration updated. */
3797 void sentinelFailoverReconfNextSlave(sentinelRedisInstance *master) {
3821         /* If too much time elapsed without the slave moving forward to
3822          * the next state, consider it reconfigured even if it is not.
3823          * Sentinels will detect the slave as misconfigured and fix its
3824          * configuration later. */
3825         if ((slave->flags & SRI_RECONF_SENT) &&
3826             (mstime() - slave->slave_reconf_sent_time) >
3827             SENTINEL_SLAVE_RECONF_TIMEOUT)
3828         {
3829             sentinelEvent(REDIS_NOTICE,"-slave-reconf-sent-timeout",slave,"%@");
3830             slave->flags &= ~SRI_RECONF_SENT;
3831             slave->flags |= SRI_RECONF_DONE;
3832         }
3833
3834         /* Nothing to do for instances that are disconnected or already
3835          * in RECONF_SENT state. */
3836         if (slave->flags & (SRI_DISCONNECTED|SRI_RECONF_SENT|SRI_RECONF_INPROG))
3837             continue;
```

这里有一个timeout检查，但是不是我们定义的failover fail, 如果该slave sentinelRedisInstance的flags表明正处于SRI_RECONF_SENT状态，并且该slave sentinelRedisInstance的slave_reconf_sent_time距今已经超过SENTINEL_SLAVE_RECONF_TIMEOUT这么长时间了，没有被提升为SRI_RECONF_xx后续状态了（slave->flags置为SRI_RECONF_SENT同更新slave->slave_reconf_sent_time几乎是同时发生的），则无论如何将slave->flags由SRI_RECONF_SENT直接提升为SRI_RECONF_DONE状态。并且产生-slave-reconf-sent-timeout这样一个失败message。fix操作留给其他逻辑,之前已经提过了。

```
/* src/sentinel.c */
3729 void sentinelFailoverDetectEnd(sentinelRedisInstance *master) {
3733     mstime_t elapsed = mstime() - master->failover_state_change_time;

3752     /* Force end of failover on timeout. */
3753     if (elapsed > master->failover_timeout) {
3754         not_reconfigured = 0;
3755         timeout = 1;
3756         sentinelEvent(REDIS_WARNING,"+failover-end-for-timeout",master,"%@ %llu",
3757             (unsigned long long) master->failover_epoch);
3758     }

3760     if (not_reconfigured == 0) {
3761         sentinelEvent(REDIS_WARNING,"+failover-end",master,"%@ %llu",
3762             (unsigned long long) master->failover_epoch);
3763
3764         master->failover_state = SENTINEL_FAILOVER_STATE_UPDATE_CONFIG;
3765         master->failover_state_change_time = mstime();
3766     }
  
3768     /* If I'm the leader it is a good idea to send a best effort SLAVEOF
3769      * command to all the slaves still not reconfigured to replicate with
3770      * the new master. */
3771     if (timeout) {
3772         dictIterator *di;
3773         dictEntry *de;
3774
3775         di = dictGetIterator(master->slaves);
3776         while((de = dictNext(di)) != NULL) {
3777             sentinelRedisInstance *slave = dictGetVal(de);
3778             int retval;
3779
3780             if (slave->flags &
3781                 (SRI_RECONF_DONE|SRI_RECONF_SENT|SRI_DISCONNECTED)) continue;
3782
3783             retval = sentinelSendSlaveOf(slave,
3784                     master->promoted_slave->addr->ip,
3785                     master->promoted_slave->addr->port);
3786             if (retval == REDIS_OK) {
3787                 sentinelEvent(REDIS_NOTICE,"+slave-reconf-sent-be",slave,"%@");
3788                 slave->flags |= SRI_RECONF_SENT;
3789             }
3790         }
3791         dictReleaseIterator(di);
3792     }  
```

sentinelFailoverDetectEnd异常的逻辑就是，如果自上次更新failover_state_change_time已经超过master->failover_timeout这么长时间，**则强制将not_reconfigured赋值为0，将该次failover按照后续的正常+failover-end逻辑处理，但是输出+failover-end-for-timeout message。这里看起来是个特殊情况,之前没有注意到，需要处理, TODO，目前为止，+failover-end-for-timeout还没在测试情况下发生过. 不过此时failover_state肯定已经到达SENTINEL_FAILOVER_STATE_RECONF_SLAVES状态，表示的是对new master的提升操作已经完成，但是old master的剩余slave还不一定被完全reconfig,不能算作是失败的failover，不能就此中断failover，对于没有完成的操作，其他逻辑后续会fix。**

另外此处还有一个best effort操作，如果是+failover-end-for-timeout的情况，则给所有不是SRI_RECONF_DONE|SRI_RECONF_SENT|SRI_DISCONNECTED状态的，发送最后一次sentinelSendSlaveOf，并标记flags为SRI_RECONF_SENT。为什么是最后一次，因为上面的逻辑是如果是timeout为true，则一定会进入+failover-end的逻辑，failover_state由SENTINEL_FAILOVER_STATE_RECONF_SLAVES提升为SENTINEL_FAILOVER_STATE_UPDATE_CONFIG状态，则此sentinelFailoverDetectEnd不会被再次重试。

**SENTINEL_FAILOVER_STATE_UPDATE_CONFIG -> SENTINEL_FAILOVER_STATE_NONE**

此阶段情况没有那么复杂，基本不会有abort的状态。


至此可以看到failover_state_change_time和failover_timeout这对孪生兄弟就是用于判断自上次failover_state_change_time更新以来是否过去了已经超过failover_timeout这么长时间，是的话，则做上诉提到的一些timeout处理。所以failover_timeout这个配置不是控制整个流程的timeout的，而是某些failover阶段的timeout。

至此，failover的主要流程已经讲完了。
