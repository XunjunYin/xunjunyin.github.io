---
layout: post
title: mem dump中unreachable objects分析
---

## 现象

* 某个大版本上线后，某服务频繁报警: load过高，但很快就会恢复，即间歇性load过高，原因难以定位。

## 观察

* 从报警系统观察报警时间分布，有一些周期性，但周期会在1小时到2小时之间，不稳定
  * 初步排除是定时任务导致
* 观察gc日志，发现报警时间点与gc时间开销较高的点比较吻合，因此着重观察GC
* 发现JVM GC相关的配置不是很合理，做了简单的优化：将old区的并行收集改为CMS，情况有所改善，但问题依然存在
* 该现象是上线后才出现的，可以认定是业务逻辑所致。着重查看了业务逻辑的diff，因业务逻辑改动较多，未能直接找到原因。
* 既然是GC所致，就先看看GC为何这么慢吧！

## 初步定位
* 用jmap做了两份内存的dump，一份是old快满（用jstat来实时观察）即即将发生full gc之前的dump，另一份是GC刚刚结束的dump
  * 这样可以明显地对二者做对比。
* 用mat打开（在机群上开了个vncserver，在本地用vncviewer连接上）两份dump,发现两份dump中reachable objects均只有400m不到，而unreachable objects有3G多
* 观察unreachable objects，按shallow heap大小排序，忽略byte和char等基本类型（复杂类型数据都是由基本类型组成的）发现TreeMap占了300M+，让人怀疑。
* 然而因为是unreachable的，无法直接查看这些TreeMap对象的上用。如何才有知道到底谁引用它们呢？在代码里grep了一下，发现自己项目中没有任何地方用到TreeMap，从页抢断应该是第三方库用到的。
* 从另外一个角度考虑，我可以先看一下reachable objects中TreeMap的引用。
* 发现其中对treeMap有引用的对象中，个数较多的是JDBC4ResultSet和ShardedJedis。先看前者在unreachable objects中的情况，共有22158个垃圾对象。
* 在mat的dominator tree中看JDBC4ResultSet的大小，可以知道每个JDBC4ResultSet的retained heap大小为4.928k（也有不同，但出入不大），从而22158个的retained heap大小为22158 * 5k 约100M，远小于heap区的大小，因此不是造成问题的原因，忽略。
* 再看后者ShardedJedis，从unreachable objects中可以看到共有4214个垃圾对象。
* 同时从dominator tree中可以看出每个ShardedJedis的Retained heap大小为323k,从而4214个的retained heap大小的4214*323k约为1.4G
* 至此，可以基本认定sharedeJedis为主要的垃圾，load间歇性过高的原因正是因为这些垃圾快速的生成和回收所致。

## 进一步分析
* 看一下我们对SharededJedisPool的配置：

		#最大分配的对象数
		redis.pool.maxActive=500
		#最大能够保持idel状态的对象数
		redis.pool.maxIdle=100
		#当池内没有返回对象时，最大等待时间
		redis.pool.maxWait=200
		#当调用borrow Object方法时，是否进行有效性检查
		redis.pool.testOnBorrow=false
		#当调用return Object方法时，是否进行有效性检查
		redis.pool.testOnReturn=false
		
* 该配置与dump中的pool配置一致。
* 通过netstat grep redisserver的端口，可以查到只有100个连接，即可预估在“够用”的情况下，redis的连接池中通常只会有100个左右（大多为空闲）
* 那么问题来了：为什么会有这么多ShardedJedis的垃圾呢？
* 我们redis连接池是用GenericObjectPool来管理的, 需要时从里面借(borrow object)，用完了再还回去(return object)。如果借的时候不够，就会new一个SharededJedis来给借用者。同时还有一个清理机制，每隔一段时间去把多余的空闲对象清理掉。从上图中可以看出，这个清理机制的间隔时间_timeBetweenEvictionRunsMills是30000ms，即30s.
* 进一步，这些ShardedJedis能度过Young GC并进入old区，必然是在young区gc时没有被清理，从而可推断该进程做一次young GC的周期会远小于GenericObjectPool的清理周期。用jstat –gctuil查看：

		[love@love21 ~]$ jstat -gcutil 21416 1000
		  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT
		  8.67   0.00  43.38  16.74  52.69  56012 2064.246   143   24.948 2089.193
		  8.67   0.00  77.59  16.74  52.69  56012 2064.246   143   24.948 2089.193
		  0.00   7.91   1.92  16.84  52.69  56013 2064.278   143   24.948 2089.225
		  0.00   7.91  34.35  16.84  52.69  56013 2064.278   143   24.948 2089.225
		  0.00   7.91  64.58  16.84  52.69  56013 2064.278   143   24.948 2089.225
		  0.00   7.91  78.19  16.84  52.69  56013 2064.278   143   24.948 2089.225
		  9.43   0.00   0.00  16.90  52.69  56014 2064.311   143   24.948 2089.258
		  9.43   0.00  22.36  16.90  52.69  56014 2064.311   143   24.948 2089.258
		  9.43   0.00  55.95  16.90  52.69  56014 2064.311   143   24.948 2089.258
		  9.43   0.00  90.86  16.90  52.69  56014 2064.311   143   24.948 2089.258
		  0.00   8.99  18.07  16.95  52.69  56015 2064.344   143   24.948 2089.292
		  0.00   8.99  51.25  16.95  52.69  56015 2064.344   143   24.948 2089.292
		  0.00   8.99  72.06  16.95  52.69  56015 2064.344   143   24.948 2089.292
		  0.00   8.99  92.20  16.95  52.69  56015 2064.344   143   24.948 2089.292
		  7.41   0.00  30.01  17.07  52.69  56016 2064.379   143   24.948 2089.326
		  7.41   0.00  65.24  17.07  52.69  56016 2064.379   143   24.948 2089.326
		  7.41   0.00  97.34  17.07  52.69  56016 2064.379   143   24.948 2089.326

* 可以看到约5s左右就会进行一次young GC，与推断相符。
* 现在，基本有了一些思路：1. 业务中有向GenericObjectPool borrow大量的object(应该远大于100)的逻辑，2. 并且很可能是定时性的任务(crontab 或 scheduleTask). 
* 结合上述两点并结合代码最新的改动，发现有地方将获取50个数据对象改为了获取250个，而为了提交性能，会用future模式开多个线程并发向一个ThreadPoolExecutor 提交大量的任务，并且该任务对redis有请求操作，而这个ThreadPoolExecutor的coreSize是200(注意，远大于redis pool的100):

		private static final ThreadPoolExecutor executor = new ThreadPoolExecutor(200, 400, 3600, TimeUnit.SECONDS, 
		        new LinkedBlockingQueue<Runnable>(500), new ThreadFactory() {
		    
		            private AtomicInteger id = new AtomicInteger(0);
		            @Override
		            public Thread newThread(Runnable r) {
		                Thread thread = new Thread(r);
		                thread.setName("Love-" + id.addAndGet(1));
		                return thread;
		            }
		        }, new ThreadPoolExecutor.CallerRunsPolicy());


* 再追溯这个50到250的改动，实际上是可以很简单的做优化的。
* 至此，问题定位到，优化并上线，不再频繁出现load过高的问题，即GC不再如此频繁

## 经验
* 解决一个性能问题好比侦察案件，可以多采集利用分析各种信息，不要死磕。虽然最终发现问题很简单，并一开始就认定问题出在业务逻辑上，可几个人花了不少时间去查代码，都没能找到原因。（该问题被一些设计模式“隐藏”的较深）
