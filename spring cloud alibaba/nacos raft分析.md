# nacos raft协议源码分析
raft一致性协议,相比于paxos协议,更容易理解和实现,nacos的cp一致性协议实现,就是基于Raft协议,主要针对以下几点进行分析
1. 选举
2. 数据一致性
3. 网络隔离产生的脑裂问题
4. 源码
## raft选举过程
##### 在raft中,任何一个服务器可能是以下三种角色之一
1. Leader:负责处理客户端请求
2. Follower:选民
3. Candidate:候选人
##### 什么时候会触发选举?
1. 服务启动时
2. leader挂了
3. follower挂了几个导致心跳或者同步Log的请求确认数小于一半
#### 先看一个正常场景下选举的过程
- 第一步:同时启动三个节点,每个节点都随机给自己设置一个倒计时的时间(15s~20s随机)	
- 第二步:每个节点会自己倒计时,所以设置15秒节点,会先变成候选人,然后向其他两个节点发送vote for me 的请求
- 第三步:其他节点接到请求后,会重置超时时间,并反馈给condidate节点,就不参加选举了.	
- 第四步:当候选人拿到3票(包含自己的一票),也就是超过半数,候选人就变成了leader,并且Leader会向各个follower节点发送心跳并重置follower节点的选举超时时间,好像是500ms一次,远远小于15~20s,正常情况下,保证Leader节点不变
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-1.png)

## 数据的同步
也是采用票数过半原则,当主节点接到一个set请求时,需要先通知所有follower节点,得到过半节点的确认后(这个确认就不包含自己),set请求才能成功返回
## 网络隔离
如图:
- 第一步:现在有5个节点,node C 已经是Leader
- 第二步:当C ,D被隔离后,如果有客户端向C发起set请求,此时C是无法工作的,因为C此时还是需要3票,才能满足过半.
- 第三步:node B被选为主,此时B认为集群总数为3,接收客户端set请求时,会得到A和B两票的确认,所以B称为真正的Leader.
- 第四步:当网络恢复后,两个Leader 会比较Term值,大的一方为Leader,C,D节点会同步数据,集群数据实现最终一致性(这里就不画图了,脑补一下吧)
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-2.png)

