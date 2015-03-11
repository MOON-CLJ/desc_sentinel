# sentinel 源码阅读指南
------------------

## 以下内容

1. **sentinel简介**

2. **sentinel failover过程的详细介绍**

3. **sentinel failover重点细节**

	sentinel之间、redis之间的所有交互的细节,各种epoch更新的细节
	
	- hello msg详解,hello msg发给sentinel干嘛,pubsub消息直接发给sentinel，sentinel能处理吗,答案在3027

	- vote的细节

	- epoch在sentinelStartFailover的作用
	
	- epoch在sentinelRefreshInstanceInfo的作用
	
	- epoch在sentinelAskMasterStateToOtherSentinels中的作用
	
	- epoch在sentinelReceiveIsMasterDownReply的作用
	
	- epoch在sentinelCommand is-master-down-by-addr部分的作用
	
4. **sentinel failover过程的各种pubsub message由来的详细介绍**

5. **配合上面的所有pubsub message现有方案的做法**

6. **遇到过的sentinel failover过程的特例以及处理方式**

7. **slave down的现有方案**

8. **redis24的master down的现有方案**

9. **其他方面的一些经验**

	- sentinel dockerize
	
	- maestro项目简介
	
	- 链接泄露，pid耗尽 

10. **现有方案的不足**

11. **sentinel现在的不足和可期待的改进**

12. **附录**
	- SRI_RECONF_*系列
				
	- master->failover_start_time会归零
	
	- sentinelVoteLeader别人和自己的投票方式是不一样的
