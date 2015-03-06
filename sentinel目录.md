# sentinel 相关经验
------------------

## 以下内容

1. **sentinel简介**

2. **sentinel failover过程的详细介绍**

	- sdown,odown
	
	- +xdown -xdown一定是配对的吗，即odown和sdown的详细介绍

3. **sentinel之间的所有交互的细节**
	
	- hello msg详解,hello msg发给sentinel干嘛,pubsub消息直接发给sentinel，sentinel能处理吗,答案在3027
	

4. **各种epoch更新的细节**

5. **sentinel failover过程的各种pubsub message由来的详细介绍**

6. **配合上面的所有pubsub message现有方案的做法**

7. **遇到过的sentinel failover过程的特例以及处理方式**

8. **slave down的现有方案**

9. **redis24的master down的现有方案**

10. **其他方面的一些经验**

	- sentinel dockerize
	
	- maestro项目简介
	
	- 链接泄露，pid耗尽 

11. **现有方案的不足**

12. **sentinel现在的不足和可期待的改进**

13. **附录**
	- SRI_RECONF_*系列
	
	- S_DOWN O_DOWN会不会出现在sentinel上
	
	- SRI_MASTER_DOWN SRI_S_DOWN SRI_O_DOWN
		
	- master->failover_start_time会归零
	
	- sentinelVoteLeader别人和自己的投票方式是不一样的






