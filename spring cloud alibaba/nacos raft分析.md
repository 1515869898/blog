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

想到一个稍微复杂一些的网络隔离场景,如果仅A和C节点无法连接,可能会出现A,C反复选举的情况.raft的解决方案是:如果C在发起选举时, B,D,E不会立即投票,需要先进性preVote投票,在preVote时,其他follow会先判断自己和Leader之间的租约有效期,如果还在有效期内,则不参与投票,preVote也就会失败,避免了Leader反复切换的问题.好像只存在于理论中,nacos 好像没有这个逻辑
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-3.png)
## 源码分析
需要先clone一份nacos server的源码https://github.com/alibaba/nacos.git我的是1.4版本
服务注册时需要同步,就从服务注册开始,com.alibaba.nacos.naming.controllers.InstanceController
```java
public String register(HttpServletRequest request) throws Exception {
     final String namespaceId = WebUtils
          .optional(request, CommonParams.NAMESPACE_ID, Constants.DEFAULT_NAMESPACE_ID);
     final String serviceName = WebUtils.required(request, CommonParams.SERVICE_NAME);
     NamingUtils.checkServiceNameFormat(serviceName);
     final Instance instance = parseInstance(request);
     serviceManager.registerInstance(namespaceId, serviceName, instance);
     return "ok";
}
```    
然后一步一步往里跟 serviceManager.registerInstance -> addInstance -> RaftConsistencyServiceImpl.put -> raftCore.signalPublish
跟进RaftCore 类的signalPublish方法.注册服务时会先判断,如果不是Leader,则找到leader并转发给Leader处理
```java
  public void signalPublish(String key, Record value) throws Exception {
        if (stopWork) {
            throw new IllegalStateException("old raft protocol already stop work");
        }
        //如果不是Leader
        if (!isLeader()) {
            ObjectNode params = JacksonUtils.createEmptyJsonNode();
            params.put("key", key);
            params.replace("value", JacksonUtils.transferToJsonNode(value));
            Map<String, String> parameters = new HashMap<>(1);
            parameters.put("key", key);
            //找到leader
            final RaftPeer leader = getLeader();
            //并转发给Leader处理
            raftProxy.proxyPostLarge(leader.ip, API_PUB, params.toString(), parameters);
            return;
        }
```
RaftCore 见类名知其意.所以在这个类中当然也可以找到leader选举的代码,在init()中.GlobalExecutor.registerMasterElection(new MasterElection()) 就是节点的倒计时操作,MasterElection会每隔500毫秒执行一次,MasterElection里会生成一个leaderDueMs 为15s~20s的随机数,每次执行会自减500. 当leaderDueMs>0 不成立时,则开始选举sendVote()
```java
 @PostConstruct
    public void init() throws Exception {
        Loggers.RAFT.info("initializing Raft sub-system");
        final long start = System.currentTimeMillis();
        //加载本地持久化的数据,db,redis...
        raftStore.loadDatums(notifier, datums);
        //term
        setTerm(NumberUtils.toLong(raftStore.loadMeta().getProperty("term"), 0L));

        Loggers.RAFT.info("cache loaded, datum count: {}, current term: {}", datums.size(), peers.getTerm());

        initialized = true;

        Loggers.RAFT.info("finish to load data from disk, cost: {} ms.", (System.currentTimeMillis() - start));
        //每500ms检查一次leader,
        masterTask = GlobalExecutor.registerMasterElection(new MasterElection());
        heartbeatTask = GlobalExecutor.registerHeartbeat(new HeartBeat());

        versionJudgement.registerObserver(isAllNewVersion -> {
            stopWork = isAllNewVersion;
            if (stopWork) {
                try {
                    shutdown();
                } catch (NacosException e) {
                    throw new NacosRuntimeException(NacosException.SERVER_ERROR, e);
                }
            }
        }, 100);

        NotifyCenter.registerSubscriber(notifier);

        Loggers.RAFT.info("timer started: leader timeout ms: {}, heart-beat timeout ms: {}",
                GlobalExecutor.LEADER_TIMEOUT_MS, GlobalExecutor.HEARTBEAT_INTERVAL_MS);
    }
```
```java
 public class MasterElection implements Runnable {

        @Override
        public void run() {
            try {
                if (stopWork) {
                    return;
                }
                if (!peers.isReady()) {
                    return;
                }

                RaftPeer local = peers.local();
                //每次自减500
                local.leaderDueMs -= GlobalExecutor.TICK_PERIOD_MS;

                if (local.leaderDueMs > 0) {
                    return;
                }

                // reset timeout
                local.resetLeaderDue();
                local.resetHeartbeatDue();
                // 当时间走完,则发起选举
                sendVote();
```
RaftPeer
```java
public void resetLeaderDue() {
        //生成一个15s~20s的随机时间
        leaderDueMs = GlobalExecutor.LEADER_TIMEOUT_MS + RandomUtils.nextLong(0, GlobalExecutor.RANDOM_MS);
    }
```
继续看sendVote(),开始走投票逻辑,先试term自增,然后voteFro设置为自己,把自己改成condidate,给所有服务发送/raft/vote 发起的票的请求,让其他节点投自己
```java
private void sendVote() {
	RaftPeer local = peers.get(NetUtils.localServer());
	Loggers.RAFT.info("leader timeout, start voting,leader: {}, term: {}", JacksonUtils.toJson(getLeader()),local.term);
	peers.reset();
	local.term.incrementAndGet();
	//投我自己
	local.voteFor = local.ip;
	//状态改为 candidate
	local.state = RaftPeer.State.CANDIDATE;
```
再去controller里找对应请求路径,RaftController.vote() -> raftCore.receivedVote ,投票的逻辑仍然还在RaftCore这个类里.判断term,如果小于另一个节点,则为其投票,并重置自己的倒计时,把自己设置为follower状态,term值也设置为成一个节点的term
```java
public synchronized RaftPeer receivedVote(RaftPeer remote) {
        if (stopWork) {
            throw new IllegalStateException("old raft protocol already stop work");
        }
        if (!peers.contains(remote)) {
            throw new IllegalStateException("can not find peer: " + remote.ip);
        }
        RaftPeer local = peers.get(NetUtils.localServer());
        //判断term
        if (remote.term.get() <= local.term.get()) {
            String msg = "received illegitimate vote" + ", voter-term:" + remote.term + ", votee-term:" + local.term;
            Loggers.RAFT.info(msg);
            //当前节点term小,则可以给另一个节点投票
            if (StringUtils.isEmpty(local.voteFor)) {
                local.voteFor = local.ip;
            }
            return local;
        }
        //重置倒计时
        local.resetLeaderDue();
        //设置为follower 节点
        local.state = RaftPeer.State.FOLLOWER;
        //投票的Ip设置为另一个节点
        local.voteFor = remote.ip;
        //自己的term设置和另一个节点term同步
        local.term.set(remote.term.get());
        Loggers.RAFT.info("vote {} as leader, term: {}", remote.ip, remote.term);
        return local;
}
```
然后回到sendVote,当/raft/vote请求返回后,开始决策perrs.decideLeader是否能成为leader.最终会得出总票数和voreFor,并且票数过半 perrs.size()/2 +1 ,最终决策出Leader,之后会进行一些数据同步等操作
```java
public RaftPeer decideLeader(RaftPeer candidate) {
        peers.put(candidate.ip, candidate);
        SortedBag ips = new TreeBag();
        int maxApproveCount = 0;
        String maxApprovePeer = null;
        for (RaftPeer peer : peers.values()) {
            if (StringUtils.isEmpty(peer.voteFor)) {
                continue;
            }
            ips.add(peer.voteFor);
            if (ips.getCount(peer.voteFor) > maxApproveCount) {
                //总票数
                maxApproveCount = ips.getCount(peer.voteFor);
                //哪个候选人
                maxApprovePeer = peer.voteFor;
            }
        }
        //胜出
        if (maxApproveCount >= majorityCount()) {
            RaftPeer peer = peers.get(maxApprovePeer);
            peer.state = RaftPeer.State.LEADER;

            if (!Objects.equals(leader, peer)) {
                leader = peer;
                ApplicationUtils.publishEvent(new LeaderElectFinishedEvent(this, leader, local()));
                Loggers.RAFT.info("{} has become the LEADER", leader.ip);
            }
        }
        return leader;
    }
```


