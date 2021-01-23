## csrf安全问题

####  为什么要做csrf防御
CSRF跨站点请求伪造(Cross—Site Request Forgery)，跟XSS攻击一样，存在巨大的危害性.攻击者盗用了你的身份，以你的名义发送恶意请求，对服务器来说这个请求是完全合法的，但是却完成了攻击者所期望的一个操作，比如以你的名义发送邮件、发消息，盗取你的账号，添加系统管理员，甚至于购买商品、虚拟货币转账等。

####  浏览器同源策略
* 只要满足以下三项相同,则可以确定两个页面是来自同一个源的:
   ``协议,域名,端口``
![avatar](https://github.com/1515869898/blog/blob/gh-pages/web/img/csrf-2.png)

####  场景演示

1. 如何发起攻击,及攻击场景
2. get 为何不安全,post就一定安全吗
3. 关于referer https://baike.baidu.com/item/HTTP_REFERER/5358396?fr=aladdin 
4. 如何应对    关闭跨域?  csrfToken;  SameSite
####  实践及源码分析
1. spring security 如何配置csrf
- [ ] spring security HttpSessionCsrfTokenRepository   
- [x] spring security CookieCsrfTokenRepository
3. CsrfFilter  执行的一个问题  org.springframework.security.web.csrf
4. LazyCsrfTokenRepository
#### 扩展
vulhub
