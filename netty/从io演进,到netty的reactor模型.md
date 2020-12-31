# 从io演进,到netty的reactor模型
## 准备工作:计算机操作系统
#### 内核空间与用户空间

![avatar](https://github.com/1515869898/blog/blob/gh-pages/netty/pic/1.png)

#### 中断--时钟中断
系统中只有一颗cpu,是怎么运行多个程序的?
![avatar](https://github.com/1515869898/blog/blob/gh-pages/netty/pic/2.png)
#### 中断--硬中断
![avatar](https://github.com/1515869898/blog/blob/gh-pages/netty/pic/3.png)
#### 中断--软中断 
1. 内核程序的作用:为操作系统内核服务暴露出接口,提供给用户程序
2. 用户态与内核态的切换
![avatar](https://github.com/1515869898/blog/blob/gh-pages/netty/pic/4.png) 
![avatar](https://github.com/1515869898/blog/blob/gh-pages/netty/pic/5.png)
#### 中断--切换带来的损耗的问题
![avatar](https://github.com/1515869898/blog/blob/gh-pages/netty/pic/6.png)

## IO演进:BIO,NIO,多路复用器
#### BIO--代码示例,寻找问题
两次阻塞:
server.accept
readLine()
![avatar](https://github.com/1515869898/blog/blob/gh-pages/netty/pic/code-1.png)

#### BIO--代码示例,寻找问题
![avatar](https://github.com/1515869898/blog/blob/gh-pages/netty/pic/code-2.png)
#### BIO--java的内核调用过程
![avatar](https://github.com/1515869898/blog/blob/gh-pages/netty/pic/code-3.png)



## netty的Reactor原理



## 基于netty实现tomcat




