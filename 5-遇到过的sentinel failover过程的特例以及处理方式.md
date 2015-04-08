## **遇到过的sentinel failover过程的特例以及处理方式**
------------------------------------------------------

本章节希望记录一些我们预料之外的事情，sentinel的行为与我们的理解或者预料不一样的情况。

首先提一下，有一点分歧是，

- **我们认为一个版本的epoch只有由一个sentinel来主导，并且**

- **一个master同一时刻只能有一个sentinel来failover，即使两个sentinel以不同的epoch的名义来
failover也不行.**

这两点其实是一点，这两个假设跟作者的理解是有分歧的。具体后续会提到.
我们不接受作者的观点的原因是，因为我们更加谨慎，希望通过以上假设，致力于摒弃掉一些极端情况,
以避免它们可能带来的不确定性.同时事实证明我们的担心并不是多余的。

### sentinelVoteLeader
----------------------

https://github.com/antirez/redis/pull/2370
这个问题在以下情况发生,

- 假设master0 down掉了

- sentinel0,sentinel1先后在很短时间内认定了master0处于odown状态。

- sentinel0 将current_epoch设置为++然后作为master0的failover epoch，假设此时current_epoch为1,
然后try-failover,然后大家除了sentinel1没有收到请求没投票之外，都投给了sentinel0.注意.

- 除sentinel1外的sentinel投票的时候，current_epoch顺带着会被更新，所以此时除了sentinel1，
其他的sentinel的current_epoch都已经更新了，成为了1。而这个大了之后的epoch会通过hello msg
广播的时候广播出去。

- sentinel1 从hello msg的渠道从other sentinel获取到了新的current_epoch.
就在这之后，sentinel1在刚刚收到的新的current_epoch即1的基础上再次++,将current_epoch加到了2，
然后作为master0的failover_epoch让other sentinel投票，其他sentinel看到这个更大的req_epoch之后，
又纷纷参与了投票，但是这次汇总投票的leader指向sentinel1，sentinel1在sentinel0之后很短时间内，
又成功获得了关于master0的failover的leader权限。

- 所以从外部观察到了sentinel1紧接着sentinel0都在failover
master0但是是以不同的epoch进行的情况。

处理的方式就是修改sentinelVoteLeader,让other sentinel在对一个master投票的时候考虑，如果
距离上次参与该master failover的投票时间太短,以至于小于一个failover_timeout的时间。
则投票给同一个leader，而不会在这么短的时间内改变自己的意见。这样做可以规避此问题。
为什么呢，因为前面的sentinel肯定获得了大多数的投票，这个大多数可能包括前面的sentinel自己,
但是不包括后面的sentinel,不然也就不会有后者自行发起failover的情况。
但是这样也就保证了后面的sentinel来要求投票的时候，这些大多数sentinel不会投给后者，
后者则不可能在短时间内成为leader。

```
/* src/sentinel.c */
char *sentinelVoteLeader(sentinelRedisInstance *master, uint64_t req_epoch, char *req_runid, uint64_t *leader_epoch) {
     {
-        sdsfree(master->leader);
-        master->leader = sdsnew(req_runid);
+        mstime_t time_since_last_vote = mstime() - master->failover_start_time;
+        if (time_since_last_vote > master->failover_timeout ||
+            strcasecmp(req_runid,server.runid) == 0 ||
+            master->leader == NULL) {
+            sdsfree(master->leader);
+            master->leader = sdsnew(req_runid);
+        }
```

### sentinelGetLeader
---------------------

https://github.com/antirez/redis/pull/2230
在sentinelGetLeader这个当前sentinel统计failover的master下的关于failover的leader的投票时,
还会有一个同前面提到的sentinelVoteLeader的行为有关联的行为会不满足我们的假设。

- 起因在于,

    ```
    /* src/sentinel.c */
    3187 void sentinelAskMasterStateToOtherSentinels(sentinelRedisInstance *master, int flags) {
    3218         retval = redisAsyncCommand(ri->cc,
    3219                     sentinelReceiveIsMasterDownReply, NULL,
    3220                     "SENTINEL is-master-down-by-addr %s %s %llu %s",
    3221                     master->addr->ip, port,
    3222                     sentinel.current_epoch,
    3223                     (master->failover_state > SENTINEL_FAILOVER_STATE_NONE) ?
    3224                     server.runid : "*");
    ```

    可以看到，对于一个已经开始的master的failover而言，SENTINEL is-master-down-by-addr命令中的
    req_epoch部分是sentinel.current_epoch，即在等待elect这段时间内，req_epoch可能会不断随着sentinel.current_epoch
    的增加而增加，而不再是failover开始时的与当初current_epoch相等的failover_epoch.

- 争议点在于,

    ```
    /* src/sentinel.c */
    3293 char *sentinelGetLeader(sentinelRedisInstance *master, uint64_t epoch) {
    3308     /* Count other sentinels votes */
    3309     di = dictGetIterator(master->sentinels);
    3310     while((de = dictNext(di)) != NULL) {
    3311         sentinelRedisInstance *ri = dictGetVal(de);
    3312         if (ri->leader != NULL && ri->leader_epoch == sentinel.current_epoch)
    3313             sentinelLeaderIncr(counters,ri->leader);
    3314     }
    ```

    上面的sentinelAskMasterStateToOtherSentinels是当前sentinel向other sentinel发起的投票请求。
    而sentinelGetLeader则是统计other sentinel的投票回应。
    为了与上面的sentinelAskMasterStateToOtherSentinels中当前req_epoch随着sentinel.current_epoch不断增加
    这一行为一致，统计的时候，sentinel的实现也是将other sentinel的ri->leader_epoch与sentinel.current_epoch
    进行比较.

所以就出现了以下情况，

- sentinel0和sentinel1同时以相同的current_epoch比如说epoch 1开始对同一个比如说master0进行failover。

- sentinel0获得了这一轮投票的大多数，然后正式对master0进行了failover。

- sentinel1没有获得大多数投票，但是并没有因为立即停止对选票的追求，即try-failover此时并没有停止。
而是等待10s的选举时间结束。

- 但是sentinel1的等待马上就会被证明是不是白费的，如果之前的sentinelVoteLeader在短时间内可以改变自己的投票选择的行为
是被允许的话.

- 因为在此种情况下，sentinel1的current_epoch增加之后比如说为2，即使master0的failover_epoch此时
还是1,但是投票请求发出的req_epoch已经是2了，此时other sentinel收到master0下的新的req epoch 2的
投票请求时，纷纷投给了sentinel1,并且反馈给sentinel1的leader_epoch也纷纷为2.

- 而sentinel1此时还在为统计failover_epoch为1的master0的failover统计投票，看是否需要继续。但是恰好
sentinelGetLeader统计投票时采用的是ri->leader_epoch == sentinel.current_epoch这样一个判定。
而此时other sentinel的ri->leader_epoch都为2,sentinel1的sentinel.current_epoch也为2。
所以如果投票大多数汇集到sentinel1上，则sentinel1的failover_epoch为1的master0的failover获得了
other sentinel的投票肯定，得以继续。而注意到sentinel0在此之前也是以epoch为1进行的failover。这与我们的假设
相悖。

所以之前提到的sentinelVoteLeader的短时间内变换vote leader的行为如果被制止，此处发生这种情况的条件就会被砍掉一部分从而
可以避免，但是与此同时sentinelGetLeader统计投票时用sentinel.current_epoch这个行为也有待商榷。
如果让other sentinel的投票回应ri->leader_epoch与failover的failover_epoch相匹配，则可以避免上述问题。
即使sentinelAskMasterStateToOtherSentinels还是采用sentinel.current_epoch作为req_epoch.

```
/* src/sentinel.c */
-        if (ri->leader != NULL && ri->leader_epoch == sentinel.current_epoch)
+        if (ri->leader != NULL && ri->leader_epoch == epoch)
```


### 期待作者对于sentinel当前“poorly desynchronized”的现状进行改进
-----------------------------------------------------------------

从上面提到的两种情况可以看到，sentinel关于投票的实现还差一点火候，我们上面提到的投票的改动是尽量不做大的改动的情况下的
临时之举，大的改动还是等待redis作者来实现吧。


### 新的处理方式，放开对sentinel的限制

- 不同的sentinel可以以不同epoch对同一个master进行连续相邻的failover。

- 不同的sentinel可以以相同的epoch对同一个master进行连续相邻的failover。

- 一个sentinel一个epoch只能有一次failover,即只对应一个master。

    在一个sentinel内部发起failover需要将全局的current_epoch++作为failover epoch,
    即不同的master的failover之间需要去sentinel内部全局注册failover_epoch.
