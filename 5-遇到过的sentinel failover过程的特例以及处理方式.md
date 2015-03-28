## **遇到过的sentinel failover过程的特例以及处理方式**
------------------------------------------------------

本章节希望记录一些我们预料之外的事情，sentinel的行为与我们的理解或者预料不一样的情况。

**首先提一下，有一点分歧是，

- 我们认为一个版本的epoch只有由一个sentinel来主导，并且

- 一个master同一时刻只能有一个sentinel来failover，即使两个sentinel以不同的epoch的名义来
failover也不行，

这两点其实是一点，这两个假设跟作者的理解是有分歧的。具体后续会提到.
我们不接受作者的观点的原因是，因为我们更加谨慎，希望通过以上假设，致力于摒弃掉一些极端情况,
以避免它们可能带来的不确定性.同时事实证明我们的担心并不是多余的。**

### sentinelGetLeader
---------------------

在sentinelGetLeader这个当前sentinel统计failover的master下的关于failover的leader的投票时

### sentinelVoteLeader
----------------------

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
