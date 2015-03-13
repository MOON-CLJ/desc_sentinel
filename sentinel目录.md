# sentinel 源码阅读指南
------------------

## 以下内容

1. **sentinel简介**

2. **sentinel failover过程的详细介绍**

3. **sentinel failover重点细节**
 	
6. **遇到过的sentinel failover过程的特例以及处理方式**
 
11. **sentinel现在的不足和可期待的改进**

-----
	
4. **sentinel failover过程的各种pubsub message由来的详细介绍**

5. **配合上面的所有pubsub message现有方案的做法**

7. **slave down的现有方案**

8. **redis24的master down的现有方案**

9. **其他方面的一些经验**

	- sentinel dockerize
	
	- maestro项目简介
	
	- 链接泄露，pid耗尽 

10. **现有方案的不足**
 
12. **附录**
					
	- master->failover_start_time会归零
	
	- sentinelVoteLeader别人和自己的投票方式是不一样的
