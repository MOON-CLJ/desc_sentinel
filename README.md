# sentinel 源码阅读指南
----------------------

## 以下内容

- **sentinel简介**

- **sentinel主逻辑函数**

- **sentinel failover过程的详细介绍**

- **sentinel failover重点细节**

- **遇到过的sentinel failover过程的特例以及处理方式**

- **sentinel现在的不足和可期待的改进**

- **sentinel failover过程的各种pubsub message由来的详细介绍**

- **配合上面的所有pubsub message现有方案的做法**

- **slave down的现有方案**

- **redis24的master down的现有方案**

- **其他方面的一些经验**

    - sentinel dockerize

    - maestro项目简介

    - 链接泄露，pid耗尽

- **现有方案的不足**

- **附录**

    - 报告的角色与当前sentinel的看法不匹配

    - SRI_MASTER SRI_SLAVE 这两个flag之间会切换么

    - redis master instance是在什么时候重新config的

    - failover过程中old master被重启会有什么影响
