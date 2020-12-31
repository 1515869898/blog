#### 场景
直销转移经纪人,需要同时更新车商库车商表,另一个库的经纪人表,最好还要写一条操作日志到商机库分库的操作日志表.需要引入分布式事务方案来解决一致性

#### 术语:
##### `TC (Transaction Coordinator) - 事务协调者`
  维护全局和分支事务的状态，驱动全局事务提交或回滚。
##### `TM (Transaction Manager) - 事务管理器`
  定义全局事务的范围：开始全局事务、提交或回滚全局事务。
##### `RM (Resource Manager) - 资源管理器`
  管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。
	
#### SEATA-AT模式优势
1. 对代码几乎零侵入,对现有事务理论上无影响,不需要额外业务代码特殊处理,开发者几乎无感知
2. seata AT模式不同于传统的两阶段协议,属于一种演变
3. 相比于TCC,对开发者友好
4. 开源社区活跃度好
#### 选型:
- TC:Seata-server
- 配置中心,服务注册与发现:Nacos-server
#### AT模式二阶工作流程:
一阶段
1. 解析SQL生成查询语句
2. 得到前镜像
3. 执行sql,更新记录(未提交)
4. 得到后镜像
5. 插入undo_log日志(未提交)
6. 提交前,RM向TC注册分支,并申请对应数据的主键记录的锁(表必须有主键)
7. 本地事务提交,undo_log提交
8. RM想TC汇报结果
9. TC最终通知全局事务结果,一阶段结束,开始二阶段全局回滚或全局提交
二阶段-回滚
1. 各个RM收到TC分支回滚请求,开启本地事务
2. 通过XID和Branch ID查找对应undo_log
3. 数据校验:拿undo_log和当前数据比较,如果不同,说明数据被当前全局事务之外动作做了修改.这种情况需要增加策略,可以转为人工处理,业务上最好选好场景,避免脏写
4. 没有出现脏写,直接还原
5. 提交本地事务,由RM上报TC
二阶段-提交
1. 收到TC分支提交请求,异步提交,RM立即上报TC
2. 异步批量删对应undo_log记录
#### SEATA AT模式的实现过程
由dealer-app发起全局事务,动态数据源的方式调用dealer-service、agent-service和rpc的方式调用scf服务,RM(相当于dataSource)将各自的本地事务执行情况汇报给TC,最终再有TC决定全局事务的提交或回滚
某个节点回滚的时候突然宕机了咋办?
server端会通过定时任务向其派发回滚请求
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/seata-1.png)

#### AT模式写隔离场景
AT模式默认设置的隔离级别是读未提交,写已提交,上面有讲过回滚过程可能发生脏写产生的原因,建议控制场景,或者加版本号的方式严格避免发生
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/seata-2.png)

#### 项目的整体架构图

![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/seata-3.png)

