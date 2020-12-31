# Nacos 服务注册原理
### Nacos 服务注册与发现需要具备的能力
1. 服务提供者把自己的协议地址注册到Nacos Server
2. 服务消费者需要从Nacos Server上去查询服务提供者提供的地址
3. Nacos Server需要感知服务提供者上下线的变化
4. 服务消费者需要动态感知到Nacos Server服务端地址的变化
### 这里用三个模型,nacos的数据模型,服务注册模型,服务发现模型,来了解一下nacos整体的设计思路
#### Nacos 数据模型
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-4.png)
- 一个application.yml是由 namespace+group+dataId确定,也就是Nacos 数据模型三元组;
- 一个集群是由namespace+group+service 决定
#### 服务注册模型
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-5.png)
服务注册模型需要关注的就是心跳机制:
1. 客户端心跳发送间隔
2. 服务端检测心跳收到心跳的超时时间(类似Netty的读间隔超时)
-  2.1 设置一个心跳超时阈值
-  2.2 记录针对于某一个服务实例最后一次更新时间
-  2.3 当前时间-当前实例最后一次更新的时间>心跳超时的阈值
#### 服务发现模型
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-6.png)
如图:每10秒拉取一次(eureka是30s),不能保证本地的实例信息的实时性;基于udp协议的push动作是可以保证实时性的,udp虽然是不可靠协议,设计者应该是认为足够满足当前场景,并且即便push偶发的失败,还可以通过每10秒的拉取来更新本地实例信息
以下便是Nacos服务发现与注册的整体设计
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-7.png)

# 关于注册的源码分析
### 1.客户端如何把自己注册到nacos server上
根据自动装配一些基本常识,先到 spring-cloud-starter-alibaba-nacos-discovery.jar 中找到spring.factories ,然后发现nacos自动装配的类
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-8.png)
打开这个类看一下,发现初始化了三个Bean:
NacosServiceRegistry,NacosRegistration,NacosAutoServiceRegistration 
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-client-code-1.png)
先打开这个NacosServiceRegistry ,很快便找到了注册服务的方法,懒得贴代码了,直接截图
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-client-code-2.png)
然后一直跟下去,最终找到了这样一个很直观的注册的post请求,就是拼一些参数如三元组参数namespaceId(传的是id),groupName,clusterName,和serviceName(应用名)等
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-client-code-3.png)
现在我们已经知道的注册的大概流程,那么服务启动时,什么时候触发的注册动作?
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-client-code-4.png)
在register上,加一个断点,然后启动服务,在idea中可以看到整个调用过程,
SimpleApplicationEventMulticaster. multicastEvent()通过beanFactory.getbean方法,获取所有到所有的容器发布事件其中就包含通继承AbstractAutoServiceRegistration并实现ApplicationListener的NacosAutoServiceRegistration(在一开始可以看到到已经被注册到beanFactory中),再用@override方法完成整个触发整个注册的流程onApplicationEvent() -> bind()->start()->register().同时还能发现一个nacoswatch,猜测是客户端的心跳
![avatar](https://github.com/1515869898/blog/blob/gh-pages/spring%20cloud%20alibaba/pic/nacos-client-code-5.png)



