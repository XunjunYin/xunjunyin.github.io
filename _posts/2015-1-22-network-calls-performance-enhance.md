---
layout: post 
title: 网络操作的性能优化
---

### 先贴上优化前后的对比图（优化于1月15日上线，19日又上线了一些细节的优化）：
* web服务器的cpu使用
![web服务器 cpu使用](http://yinxj.qiniudn.com/front.cpu.png)
* web服务器的load 
![web服务器 load](http://yinxj.qiniudn.com/front.load.png)
* rpc的连接数（主要是与redis的连接） 
![rpc的连接数](http://yinxj.qiniudn.com/rpc.tpc.num.png)
* redis服务的cpu使用
![redis服务的cpu使用](http://yinxj.qiniudn.com/redis.cpu.png)
* redis所在机器的连接数
![redis所在机器的连接数](http://yinxj.qiniudn.com/redis.tcp.num.png)

### 现象
* 某产品由于某些因素（运营，版本更新及体验等）使得活跃用户及停留时间不断提高，进而使得服务器资源使用增长，原先设定的报警阈值频繁被触发，具体表现为：
  * web机器的load与cpu使用都变高
  * 依赖的缓存服务redis连接数与cpu不断攀升
  * rpc(用于业务分离)的连接数不断攀升
  
### 如何解决
* 负载均衡，增加服务器——最简单粗暴且有效的方式
* 找到系统性能瓶颈，优化之——这是最经济也最有挑战的事
* ——考虑到线上服务尚未出现功能性问题，先尝试第二种方式

### 定位瓶颈
* 参考我之前的blog，对线上各服务（均为java进程）取jstack并用stackAnalysis工具分析其瓶颈，发现许多线程都停留在对redis的操作上
* 很显然，redis的大量请求是瓶颈

### 分析
* 仔细分析stack dump中有相似stackstrace的线程，发现对缓存的操作有不少shit的逻辑：
  * "同样的数据取两次"——如：interceptor中会对同一个user对象获取两次
  		
  		e.g.:
  		User user = userRpcServer.getUser(uid);
  		// 一些业务逻辑
  		boolean isForbidderUser = userRpcServer.isForbidderUser(uid);
  		
  	* 从上面看并没有直接问题，但仔细分析会发现userRpcServer.isForbidderUser(uid)中还会调用一次getUuser(id)
  * "取到了数据却不用"——如：获取User对象时同时获取其帐启余额，但只有与帐户相关的请求才需要获取余额，大多数接口不需要
  
  		UserAccount userAccount = userRpcServer.getUserAccount(uid);
  		UserInfo userInfo = userRpcServer.getUserInfo(uid);
  		User user =  makeUser(userAccount, userInfo);
  		
  * "已经知道绥存中不存在数据了，却还去取一次"——因为调用栈较长，所以隐藏的比较深，暂不举例。但也正因为隐藏的较深，才造成了资源的无用浪费
  * "有大量用for循环对redis做网络操作的逻辑"——如：根据userIds获取users
  
  		for(uid : uids) {
  			users.add(redisServer.getUser(uid))
  		}
* 分析redis slow log, 找出其中耗时和频繁的操作（尤其是删除操作），发现有一些优化的空间——此前已做过一次slow log对应key的优化，所以这次在这方面没做太多事情。
  		
  		
### 解决
* 对应上面分析到的问题，分别的解决方案为：
  * 同样的数据取两次 —— 相同的数据只取一次
  * 取到了数据却不用 —— 不用则不取
  * 已经知道绥存中不存在数据了，却还去取一次 —— 有的放矢
  * 有大量用for循环对redis做网络操作的逻辑 —— 尽量用批量请求
    * redis是一个集群，因此需要用ShardedJedisPipeline来做批量请求
* 优化之后报警短信完成消失。

### 优化结果评估
* 找到优化方法并进行优化之后，分别对各个优化做性能测试
* 需要对每一步优化都做性能测试，知悉各改动优化的力度。

### 思考
* 人总是：1. 犯错；2. 选择最合适的办法解决当前需求——不论是coding还是产品架构或者公司运转。
* 错误总比较明显容易解决，但随着产品发展时间推移，一些曾合理的地方会变得不太合理，甚至会起到了阻碍作用。
* 我们需要不断从现状中找到不合理的地方，并改进它，而不是盲从、习惯或停留于批判。
