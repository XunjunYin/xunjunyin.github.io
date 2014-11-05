---
layout: post
title: deadlock of threadPoolExecutor - ThreadPoolExecutor使用错误导致web服务器卡死
---

## 背景
 * 10月2号凌晨12:08收到报警，所有请求失败，处于完全不可用状态
 * 应用服务器共四台resin，resin之前由四台nginx做负载均衡

## 服务器现象及故障恢复步骤
* 登入服务器，观察resin进程，初看无任何异常，且占用资源正常，有非业务逻辑相关（一些schedule task）的日志输出，但无业务逻辑相关的日志。
  * 表明resin服务器没有在处理（新的）用户的请求
* 重启resin，并观察日志，发现resin开始处理业务，基本恢复
  * 表明重启可以解决问题
* 继续依次重启剩余的三台resin，并在重启最后一台resin之前取jstack以供分析故障原因
  * jstack可以较好的反映进程正在进行的所有逻辑，可以有效的帮助定位问题，并且耗时较少通常5s内便可完成
  * 更全面的获取进程信息的方法是取heap dump, 但耗时相对长一些，且分析时没有jstack直观（当然，由dump也可以得到jstack）
  * 若在进程有问题时直接重启而不取jstack很可能会**丧失定位问题的唯一时机** —— 此时已经基本可认定原因在resin进程内部，因此更有必要取jstack
  * 在重启第四台resin时才取jstack的原因是应该**尽快恢复线上服务可用**，因此重启前三台resin之前不应该做浪费时间的工作。但它们正常工作以后，线上的请求压力可以先由它们来负荷，因此可以对第四台进行一些快速并重要的操作后再重启
* 观察，服务基本恢复
* 看jstack文件，试图找到线索
* 突然又收到报警，发现四台resin服务又恢复原来的故障状态，所有请求均失败。于是再次重启前两台resin
* 同时到nginx处观察access log，发现所有请求均为502 bad gateway或timeout
  * 表明是nginx与resin之间的请求转发有些问题，猜测像是请求没发送给resin或是resin直接拒绝执行
* 未确定具体原因，于是尝试直接依次重启四台nginx, 并再重启剩余的resin
* 服务再次恢复，观察一小时多后也不再出现问题
* 至此，认为故障已经恢复

## 疑问
* resin进程恢复后为何又会再次故障？
* 最后一次上线是9月29号下午，如果是业务实现的问题，为何这之间一直没出问题？
* 重启nginx究竟影响了什么？难道nginx与resin内部会存在什么神奇的逻辑关联？

## 定位分析
* 对服务异常时取的jstack进行分析(在机群上用我之前写的一个stack分析脚本stackAnalysis，在命令行直接执行stackAnalysis即可), 发现resin的业务线程部分的summary如下:

	    name=resin-port-*****-*****
		threadsNum=257
		hasUserThread=true
		blockNum=0
		execNum=0
		-------------------------------------------- threads groups with different state, size=4, start --------------------------------------------
		219 threads at (state = WAITING,
		locks_waiting = [0x00000006ed230e60, 0x00000006f0b19db0, ..., 0x00000006ec643d40, 0x00000006ed09ca38, 0x00000006eb0e8600, 0x00000006f1cf2118]) :
		"resin-port-*-*" daemon prio=* tid=******** nid=******** waiting on condition [********]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <********> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService.getFilteredUserId(HomeService.java:933)
			at com.xxxxxx.productfront.service.HomeService.requestLatestTrends(HomeService.java:513)
			at com.xxxxxx.productfront.service.HomeService.getNewHomeDataFlow(HomeService.java:140)
			at com.xxxxxx.productfront.web.restful.HomeApiController.newInfo(HomeApiController.java:204)
			at com.xxxxxx.productfront.web.restful.HomeApiController$$FastClassByCGLIB$$5502b6f1.invoke(<generated>)
			at net.sf.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
			at org.springframework.aop.framework.Cglib2AopProxy$CglibMethodInvocation.invokeJoinpoint(Cglib2AopProxy.java:689)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:150)
			at org.springframework.validation.beanvalidation.MethodValidationInterceptor.invoke(MethodValidationInterceptor.java:93)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:172)
			at org.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor.invoke(MethodBeforeAdviceInterceptor.java:50)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:172)
			at org.springframework.aop.framework.Cglib2AopProxy$DynamicAdvisedInterceptor.intercept(Cglib2AopProxy.java:622)
			at com.xxxxxx.productfront.web.restful.HomeApiController$$EnhancerByCGLIB$$80fdc5c.newInfo(<generated>)
			at sun.reflect.GeneratedMethodAccessor1384.invoke(Unknown Source)
			at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
			at java.lang.reflect.Method.invoke(Method.java:597)
			at org.springframework.web.bind.annotation.support.HandlerMethodInvoker.invokeHandlerMethod(HandlerMethodInvoker.java:176)
			at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.invokeHandlerMethod(AnnotationMethodHandlerAdapter.java:436)
			at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.handle(AnnotationMethodHandlerAdapter.java:424)
			at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:923)
			at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:852)
			at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:882)
			at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:789)
			at javax.servlet.http.HttpServlet.service(HttpServlet.java:159)
			at javax.servlet.http.HttpServlet.service(HttpServlet.java:97)
			at com.caucho.server.dispatch.ServletFilterChain.doFilter(ServletFilterChain.java:109)
			at com.xxxxxx.productfront.web.filter.FlagFilter.doFilter(FlagFilter.java:56)
			at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:89)
			at com.xxxxxx.productfront.web.filter.UserInfoFilter.doFilter(UserInfoFilter.java:37)
			at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:89)
			at com.caucho.server.webapp.WebAppListenerFilterChain.doFilter(WebAppListenerFilterChain.java:114)
			at com.caucho.server.webapp.WebAppFilterChain.doFilter(WebAppFilterChain.java:156)
			at com.caucho.server.webapp.AccessLogFilterChain.doFilter(AccessLogFilterChain.java:95)
			at com.caucho.server.dispatch.ServletInvocation.service(ServletInvocation.java:289)
			at com.caucho.server.http.HttpRequest.handleRequest(HttpRequest.java:838)
			at com.caucho.network.listen.TcpSocketLink.dispatchRequest(TcpSocketLink.java:1341)
			at com.caucho.network.listen.TcpSocketLink.handleRequest(TcpSocketLink.java:1297)
			at com.caucho.network.listen.TcpSocketLink.handleRequestsImpl(TcpSocketLink.java:1281)
			at com.caucho.network.listen.TcpSocketLink.handleRequests(TcpSocketLink.java:1189)
			at com.caucho.network.listen.TcpSocketLink.handleAcceptTaskImpl(TcpSocketLink.java:985)
			at com.caucho.network.listen.ConnectionTask.runThread(ConnectionTask.java:117)
			at com.caucho.network.listen.ConnectionTask.run(ConnectionTask.java:93)
			at com.caucho.network.listen.SocketLinkThreadLauncher.handleTasks(SocketLinkThreadLauncher.java:169)
			at com.caucho.network.listen.TcpSocketAcceptThread.run(TcpSocketAcceptThread.java:61)
			at com.caucho.env.thread2.ResinThread2.runTasks(ResinThread2.java:173)
			at com.caucho.env.thread2.ResinThread2.run(ResinThread2.java:118)

		24 threads at (state = WAITING,
		locks_waiting = [0x00000006e56c9970, 0x00000006e56c6fd8, 0x00000006e28b14a8, 0x00000006e61dc9a0, 0x00000006e620e9c0, 0x00000006e3bf0648, 0x00000006e56d39e0, 0x00000006e6a051a0, 0x00000006e29fd0f0, 0x00000006e3b03728, 0x00000006e613b090, 0x00000006e5a67fc8, 0x00000006e7de7d90, 0x00000006e693a338, 0x00000006e46fc7c0, 0x00000006e28f41a0, 0x00000006e614a990, 0x00000006e61203d0, 0x00000006e45bd5f8, 0x00000006e5a65e90, 0x00000006e611feb0, 0x00000006e51d3b80, 0x00000006e4f23348, 0x00000006e692f150]) :
		"resin-port-*-*" daemon prio=* tid=******** nid=******** waiting on condition [********]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <********> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService.getNewHomeDataFlow(HomeService.java:268)
			at com.xxxxxx.productfront.web.restful.HomeApiController.newInfo(HomeApiController.java:204)
			at com.xxxxxx.productfront.web.restful.HomeApiController$$FastClassByCGLIB$$5502b6f1.invoke(<generated>)
			at net.sf.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
			at org.springframework.aop.framework.Cglib2AopProxy$CglibMethodInvocation.invokeJoinpoint(Cglib2AopProxy.java:689)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:150)
			at org.springframework.validation.beanvalidation.MethodValidationInterceptor.invoke(MethodValidationInterceptor.java:93)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:172)
			at org.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor.invoke(MethodBeforeAdviceInterceptor.java:50)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:172)
			at org.springframework.aop.framework.Cglib2AopProxy$DynamicAdvisedInterceptor.intercept(Cglib2AopProxy.java:622)
			at com.xxxxxx.productfront.web.restful.HomeApiController$$EnhancerByCGLIB$$80fdc5c.newInfo(<generated>)
			at sun.reflect.GeneratedMethodAccessor1384.invoke(Unknown Source)
			at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
			at java.lang.reflect.Method.invoke(Method.java:597)
			at org.springframework.web.bind.annotation.support.HandlerMethodInvoker.invokeHandlerMethod(HandlerMethodInvoker.java:176)
			at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.invokeHandlerMethod(AnnotationMethodHandlerAdapter.java:436)
			at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.handle(AnnotationMethodHandlerAdapter.java:424)
			at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:923)
			at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:852)
			at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:882)
			at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:789)
			at javax.servlet.http.HttpServlet.service(HttpServlet.java:159)
			at javax.servlet.http.HttpServlet.service(HttpServlet.java:97)
			at com.caucho.server.dispatch.ServletFilterChain.doFilter(ServletFilterChain.java:109)
			at com.xxxxxx.productfront.web.filter.FlagFilter.doFilter(FlagFilter.java:56)
			at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:89)
			at com.xxxxxx.productfront.web.filter.UserInfoFilter.doFilter(UserInfoFilter.java:37)
			at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:89)
			at com.caucho.server.webapp.WebAppListenerFilterChain.doFilter(WebAppListenerFilterChain.java:114)
			at com.caucho.server.webapp.WebAppFilterChain.doFilter(WebAppFilterChain.java:156)
			at com.caucho.server.webapp.AccessLogFilterChain.doFilter(AccessLogFilterChain.java:95)
			at com.caucho.server.dispatch.ServletInvocation.service(ServletInvocation.java:289)
			at com.caucho.server.http.HttpRequest.handleRequest(HttpRequest.java:838)
			at com.caucho.network.listen.TcpSocketLink.dispatchRequest(TcpSocketLink.java:1341)
			at com.caucho.network.listen.TcpSocketLink.handleRequest(TcpSocketLink.java:1297)
			at com.caucho.network.listen.TcpSocketLink.handleRequestsImpl(TcpSocketLink.java:1281)
			at com.caucho.network.listen.TcpSocketLink.handleRequests(TcpSocketLink.java:1189)
			at com.caucho.network.listen.TcpSocketLink.handleAcceptTaskImpl(TcpSocketLink.java:985)
			at com.caucho.network.listen.ConnectionTask.runThread(ConnectionTask.java:117)
			at com.caucho.network.listen.ConnectionTask.run(ConnectionTask.java:93)
			at com.caucho.network.listen.SocketLinkThreadLauncher.handleTasks(SocketLinkThreadLauncher.java:169)
			at com.caucho.network.listen.TcpSocketAcceptThread.run(TcpSocketAcceptThread.java:61)
			at com.caucho.env.thread2.ResinThread2.runTasks(ResinThread2.java:173)
			at com.caucho.env.thread2.ResinThread2.run(ResinThread2.java:118)

		13 threads at (state = WAITING,
		locks_waiting = [0x00000006eba80d68, 0x00000006eba87ff0, 0x00000006eb013438, 0x00000006eba88088, 0x00000006eb9ea590, 0x00000006eb851ea8, 0x00000006eb851f40, 0x00000006eaecb790, 0x00000006eb851d20, 0x00000006eaecb190, 0x00000006eba8ea88, 0x00000006eb8b3b58, 0x00000006eb8b3a18]) :
		"resin-port-*-*" daemon prio=* tid=******** nid=******** waiting on condition [********]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <********> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService.getFilteredUserId(HomeService.java:933)
			at com.xxxxxx.productfront.service.HomeService.requestLatestTrends(HomeService.java:513)
			at com.xxxxxx.productfront.service.HomeService.getNewHomeDataFlow(HomeService.java:140)
			at com.xxxxxx.productfront.web.restful.HomeApiController.newInfo(HomeApiController.java:204)
			at com.xxxxxx.productfront.web.restful.HomeApiController$$FastClassByCGLIB$$5502b6f1.invoke(<generated>)
			at net.sf.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
			at org.springframework.aop.framework.Cglib2AopProxy$CglibMethodInvocation.invokeJoinpoint(Cglib2AopProxy.java:689)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:150)
			at org.springframework.validation.beanvalidation.MethodValidationInterceptor.invoke(MethodValidationInterceptor.java:93)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:172)
			at org.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor.invoke(MethodBeforeAdviceInterceptor.java:50)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:172)
			at org.springframework.aop.framework.Cglib2AopProxy$DynamicAdvisedInterceptor.intercept(Cglib2AopProxy.java:622)
			at com.xxxxxx.productfront.web.restful.HomeApiController$$EnhancerByCGLIB$$80fdc5c.newInfo(<generated>)
			at sun.reflect.GeneratedMethodAccessor1384.invoke(Unknown Source)
			at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
			at java.lang.reflect.Method.invoke(Method.java:597)
			at org.springframework.web.bind.annotation.support.HandlerMethodInvoker.invokeHandlerMethod(HandlerMethodInvoker.java:176)
			at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.invokeHandlerMethod(AnnotationMethodHandlerAdapter.java:436)
			at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.handle(AnnotationMethodHandlerAdapter.java:424)
			at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:923)
			at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:852)
			at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:882)
			at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:789)
			at javax.servlet.http.HttpServlet.service(HttpServlet.java:159)
			at javax.servlet.http.HttpServlet.service(HttpServlet.java:97)
			at com.caucho.server.dispatch.ServletFilterChain.doFilter(ServletFilterChain.java:109)
			at com.xxxxxx.productfront.web.filter.FlagFilter.doFilter(FlagFilter.java:56)
			at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:89)
			at com.xxxxxx.productfront.web.filter.UserInfoFilter.doFilter(UserInfoFilter.java:37)
			at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:89)
			at com.caucho.server.webapp.WebAppListenerFilterChain.doFilter(WebAppListenerFilterChain.java:114)
			at com.caucho.server.webapp.WebAppFilterChain.doFilter(WebAppFilterChain.java:156)
			at com.caucho.server.webapp.AccessLogFilterChain.doFilter(AccessLogFilterChain.java:95)
			at com.caucho.server.dispatch.ServletInvocation.service(ServletInvocation.java:289)
			at com.caucho.server.http.HttpRequest.handleRequest(HttpRequest.java:838)
			at com.caucho.network.listen.TcpSocketLink.dispatchRequest(TcpSocketLink.java:1341)
			at com.caucho.network.listen.TcpSocketLink.handleRequest(TcpSocketLink.java:1297)
			at com.caucho.network.listen.TcpSocketLink.handleRequestsImpl(TcpSocketLink.java:1281)
			at com.caucho.network.listen.TcpSocketLink.handleRequests(TcpSocketLink.java:1189)
			at com.caucho.network.listen.TcpSocketLink.handleAcceptTaskImpl(TcpSocketLink.java:988)
			at com.caucho.network.listen.ConnectionTask.runThread(ConnectionTask.java:117)
			at com.caucho.network.listen.ConnectionTask.run(ConnectionTask.java:93)
			at com.caucho.network.listen.SocketLinkThreadLauncher.handleTasks(SocketLinkThreadLauncher.java:169)
			at com.caucho.network.listen.TcpSocketAcceptThread.run(TcpSocketAcceptThread.java:61)
			at com.caucho.env.thread2.ResinThread2.runTasks(ResinThread2.java:173)
			at com.caucho.env.thread2.ResinThread2.run(ResinThread2.java:118)

		1 threads at (state = WAITING,
		locks_waiting = [0x00000006e2b747e0]) :
		"resin-port-*-*" daemon prio=* tid=******** nid=******** waiting on condition [********]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <********> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService.getNewHomeDataFlow(HomeService.java:270)
			at com.xxxxxx.productfront.web.restful.HomeApiController.newInfo(HomeApiController.java:204)
			at com.xxxxxx.productfront.web.restful.HomeApiController$$FastClassByCGLIB$$5502b6f1.invoke(<generated>)
			at net.sf.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
			at org.springframework.aop.framework.Cglib2AopProxy$CglibMethodInvocation.invokeJoinpoint(Cglib2AopProxy.java:689)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:150)
			at org.springframework.validation.beanvalidation.MethodValidationInterceptor.invoke(MethodValidationInterceptor.java:93)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:172)
			at org.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor.invoke(MethodBeforeAdviceInterceptor.java:50)
			at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:172)
			at org.springframework.aop.framework.Cglib2AopProxy$DynamicAdvisedInterceptor.intercept(Cglib2AopProxy.java:622)
			at com.xxxxxx.productfront.web.restful.HomeApiController$$EnhancerByCGLIB$$80fdc5c.newInfo(<generated>)
			at sun.reflect.GeneratedMethodAccessor1384.invoke(Unknown Source)
			at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
			at java.lang.reflect.Method.invoke(Method.java:597)
			at org.springframework.web.bind.annotation.support.HandlerMethodInvoker.invokeHandlerMethod(HandlerMethodInvoker.java:176)
			at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.invokeHandlerMethod(AnnotationMethodHandlerAdapter.java:436)
			at org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter.handle(AnnotationMethodHandlerAdapter.java:424)
			at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:923)
			at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:852)
			at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:882)
			at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:789)
			at javax.servlet.http.HttpServlet.service(HttpServlet.java:159)
			at javax.servlet.http.HttpServlet.service(HttpServlet.java:97)
			at com.caucho.server.dispatch.ServletFilterChain.doFilter(ServletFilterChain.java:109)
			at com.xxxxxx.productfront.web.filter.FlagFilter.doFilter(FlagFilter.java:56)
			at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:89)
			at com.xxxxxx.productfront.web.filter.UserInfoFilter.doFilter(UserInfoFilter.java:37)
			at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:89)
			at com.caucho.server.webapp.WebAppListenerFilterChain.doFilter(WebAppListenerFilterChain.java:114)
			at com.caucho.server.webapp.WebAppFilterChain.doFilter(WebAppFilterChain.java:156)
			at com.caucho.server.webapp.AccessLogFilterChain.doFilter(AccessLogFilterChain.java:95)
			at com.caucho.server.dispatch.ServletInvocation.service(ServletInvocation.java:289)
			at com.caucho.server.http.HttpRequest.handleRequest(HttpRequest.java:838)
			at com.caucho.network.listen.TcpSocketLink.dispatchRequest(TcpSocketLink.java:1341)
			at com.caucho.network.listen.TcpSocketLink.handleRequest(TcpSocketLink.java:1297)
			at com.caucho.network.listen.TcpSocketLink.handleRequestsImpl(TcpSocketLink.java:1281)
			at com.caucho.network.listen.TcpSocketLink.handleRequests(TcpSocketLink.java:1189)
			at com.caucho.network.listen.TcpSocketLink.handleAcceptTaskImpl(TcpSocketLink.java:985)
			at com.caucho.network.listen.ConnectionTask.runThread(ConnectionTask.java:117)
			at com.caucho.network.listen.ConnectionTask.run(ConnectionTask.java:93)
			at com.caucho.network.listen.SocketLinkThreadLauncher.handleTasks(SocketLinkThreadLauncher.java:169)
			at com.caucho.network.listen.TcpSocketAcceptThread.run(TcpSocketAcceptThread.java:61)
			at com.caucho.env.thread2.ResinThread2.runTasks(ResinThread2.java:173)
			at com.caucho.env.thread2.ResinThread2.run(ResinThread2.java:118)

		----------------------------------------------- threads groups with different state, size=4, end -------------------------------------------


* 查看resin的配置，可以知道我们配置的最大业务线程数为256:

	    \#Throttle the number of active threads for a port
	    port_thread_max   : 256
	    accept_thread_max : 32
	    accept_thread_min : 4

* 同时从上面的stack summary结果中可以看到有257个业务逻辑的线程(以resin-port-开头的线程组)处于waiting状态，说明此时'''所有业务线程均在waiting状态，从而新的业务请求自然会被resin拒绝，造成服务不可用'''
* 从stack上可以看到所有waiting状态的业务线程都是客户端的首页接口【	at com.xxxxxx.productfront.web.restful.HomeApiController.newInfo(HomeApiController.java:204)
】，并且都是在对一个Future任务进行FutureTask.get()操作，同时由栈顶可以看出改这些线程都是在对一个队列进行操作时无法继续而使得线程waiting的：
		- parking to wait for  <********> (a java.util.concurrent.FutureTask$Sync)
		at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
		at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
		at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
		at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)

* 新的疑问产生了：这257个线程究竟在waiting什么？怎么可以这样占着xx不xx！！！
* 查看业务代码HomeApiController.java相关逻辑发现该业务逻辑（获取客户端首页数据，包括附近的人，推荐的人，新的动态等）为了提升性能，将没有耦合的逻辑用future模式来处理以实现并行执行。同时为了节省线程创建销毁的开销，类中new了一个static的ThreadPoolExecutor用于接受各类futureTask的执行：
	 	 protected static final ThreadPoolExecutor executor = new ThreadPoolExecutor(20, 100, 2, TimeUnit.MINUTES, new LinkedBlockingQueue<Runnable>(),
	            new ThreadFactory() {
	                private AtomicInteger id = new AtomicInteger(0);

	                @Override
	                public Thread newThread(Runnable r) {
	                    Thread thread = new Thread(r);
	                    thread.setName("home-service-" + id.addAndGet(1));
	                    return thread;
	                }
	            }, new ThreadPoolExecutor.CallerRunsPolicy());

且其构造函数相关含义为：


	ThreadPoolExecutor
	public ThreadPoolExecutor(int corePoolSize,
	                          int maximumPoolSize,
	                          long keepAliveTime,
	                          TimeUnit unit,
	                          BlockingQueue<Runnable> workQueue,
	                          ThreadFactory threadFactory,
	                          RejectedExecutionHandler handler)
	Creates a new ThreadPoolExecutor with the given initial parameters.
	Parameters:
	corePoolSize - the number of threads to keep in the pool, even if they are idle.
	maximumPoolSize - the maximum number of threads to allow in the pool.
	keepAliveTime - when the number of threads is greater than the core, this is the maximum time that excess idle threads will wait for new tasks before terminating.
	unit - the time unit for the keepAliveTime argument.
	workQueue - the queue to use for holding tasks before they are executed. This queue will hold only the Runnable tasks submitted by the execute method.
	threadFactory - the factory to use when the executor creates a new thread.
	handler - the handler to use when execution is blocked because the thread bounds and queue capacities are reached.
	Throws:
	IllegalArgumentException - if corePoolSize or keepAliveTime less than zero, or if maximumPoolSize less than or equal to zero, or if corePoolSize greater than maximumPoolSize.
	NullPointerException - if workQueue or threadFactory or handler are null.
	
* 因为jstack有不少线程与队列有关，为明确其构造函数中workQueue的使用，查看[ThreadPoolExecutor的java doc](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ThreadPoolExecutor.html):

		Queuing
	    Any BlockingQueue may be used to transfer and hold submitted tasks. The use of this queue interacts with pool sizing:
	        * If fewer than corePoolSize threads are running, the Executor always prefers adding a new thread rather than queuing.
	        * If corePoolSize or more threads are running, the Executor always prefers queuing a request rather than adding a new thread.
	        * If a request cannot be queued, a new thread is created unless this would exceed maximumPoolSize, in which case, the task will be rejected.
	    There are three general strategies for queuing:
	        Direct handoffs. A good default choice for a work queue is a SynchronousQueue that hands off tasks to threads without otherwise holding them. Here, an attempt to queue a task will fail if no threads are immediately available to run it, so a new thread will be constructed. This policy avoids lockups when handling sets of requests that might have internal dependencies. Direct handoffs generally require unbounded maximumPoolSizes to avoid rejection of new submitted tasks. This in turn admits the possibility of unbounded thread growth when commands continue to arrive on average faster than they can be processed.
	        Unbounded queues. Using an unbounded queue (for example a LinkedBlockingQueue without a predefined capacity) will cause new tasks to wait in the queue when all corePoolSize threads are busy. Thus, no more than corePoolSize threads will ever be created. (And the value of the maximumPoolSize therefore doesn't have any effect.) This may be appropriate when each task is completely independent of others, so tasks cannot affect each others execution; for example, in a web page server. While this style of queuing can be useful in smoothing out transient bursts of requests, it admits the possibility of unbounded work queue growth when commands continue to arrive on average faster than they can be processed.
	        Bounded queues. A bounded queue (for example, an ArrayBlockingQueue) helps prevent resource exhaustion when used with finite maximumPoolSizes, but can be more difficult to tune and control. Queue sizes and maximum pool sizes may be traded off for each other: Using large queues and small pools minimizes CPU usage, OS resources, and context-switching overhead, but can lead to artificially low throughput. If tasks frequently block (for example if they are I/O bound), a system may be able to schedule time for more threads than you otherwise allow. Use of small queues generally requires larger pool sizes, which keeps CPUs busier but may encounter unacceptable scheduling overhead, which also decreases throughput.

* LinkedBlockingQueue是没有上限的队列，由undounded queues的说明可知因为它的使用会导致maximumPoolSize的设置失效，即'''构造函数中传入的100是没有意义的'''，只会有corePoolSize大小的线程数，也就是我们设置的20个线程存活在pool中。很明显，这样的结果与我们如此使用的预期是不相符的。
* 发现了一些问题，上面的疑问也清晰了一些：很可能那257个线程是在等待ThreadPoolExecutor去完成任务，但由于只有20个线程在干活，因此这257个线程的futureTask被放到了LinkedBlockingQueue中，只有那20个线程做完手中的事之后这257个线程对应的futureTask才有机会被执行。
* 于是，现在的疑问变成了：这20个线程究竟在做什么？虽然只有20个，但服务正常的时候它们应该非常快才对（不然问题早就出现了），但现在为什么那么慢以至于block住了所有的业务线程，'''它们究竟在做做什么？'''
* 再回头看stack summary的结果，发现确实有一个名为home-service-*的线程组，并且一共20个线程，这与上面100的maximumPoolSize的设置是无效的分析一致。看一下这个线程组各线程的具体状态：

		name=home-service-**
		threadsNum=20
		hasUserThread=true
		blockNum=0
		execNum=0
		-------------------------------------------- threads groups with different state, size=3, start --------------------------------------------
		10 threads at (state = WAITING,
		locks_waiting = [0x00000006e59d3fe8, 0x00000006e4e848d0, 0x00000006e598b410, 0x00000006e4f90898, 0x00000006e674a510, 0x00000006e4e841b8, 0x00000006e4e845a0, 0x00000006e674a1e0, 0x00000006e4f90fc0, 0x00000006e4f90c90]) :
		"home-service-*" daemon prio=* tid=******** nid=******** waiting on condition [********]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <********> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService.getExtendedTrend(HomeService.java:625)
			at com.xxxxxx.productfront.service.HomeService.buildTrend(HomeService.java:561)
			at com.xxxxxx.productfront.service.HomeService.access$000(HomeService.java:69)
			at com.xxxxxx.productfront.service.HomeService$2.call(HomeService.java:148)
			at com.xxxxxx.productfront.service.HomeService$2.call(HomeService.java:143)
			at java.util.concurrent.FutureTask$Sync.innerRun(FutureTask.java:303)
			at java.util.concurrent.FutureTask.run(FutureTask.java:138)
			at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
			at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
			at java.lang.Thread.run(Thread.java:662)

		8 threads at (state = WAITING,
		locks_waiting = [0x00000006e692a260, 0x00000006e5027360, 0x00000006e5027428, 0x00000006e4f912f0, 0x00000006e4f913d8, 0x00000006e598b740, 0x00000006e598b808, 0x00000006e4f90bc8]) :
		"home-service-*" daemon prio=* tid=******** nid=******** waiting on condition [********]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <********> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService.getExtendedRecommendInfo(HomeService.java:740)
			at com.xxxxxx.productfront.service.HomeService.buildRecommend(HomeService.java:453)
			at com.xxxxxx.productfront.service.HomeService$3.call(HomeService.java:242)
			at com.xxxxxx.productfront.service.HomeService$3.call(HomeService.java:237)
			at java.util.concurrent.FutureTask$Sync.innerRun(FutureTask.java:303)
			at java.util.concurrent.FutureTask.run(FutureTask.java:138)
			at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
			at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
			at java.lang.Thread.run(Thread.java:662)

		2 threads at (state = WAITING,
		locks_waiting = [0x00000006e4e844e8, 0x00000006e4e84100]) :
		"home-service-*" daemon prio=* tid=******** nid=******** waiting on condition [********]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <********> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService.getExtendedNearByInfo(HomeService.java:876)
			at com.xxxxxx.productfront.service.HomeService.buildNearBy(HomeService.java:476)
			at com.xxxxxx.productfront.service.HomeService$4.call(HomeService.java:258)
			at com.xxxxxx.productfront.service.HomeService$4.call(HomeService.java:253)
			at java.util.concurrent.FutureTask$Sync.innerRun(FutureTask.java:303)
			at java.util.concurrent.FutureTask.run(FutureTask.java:138)
			at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
			at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
			at java.lang.Thread.run(Thread.java:662)

		----------------------------------------------- threads groups with different state, size=3, end -------------------------------------------

* 仔细观察stacktrace, 可以发现有个很奇怪的地方：这些线程都是从一个FutureTask.run开始，然后又调用了另一个FutureTask，并waiting在FutureTask.get()方法。
* 也就是说，我们的代码逻辑中，在FutureTask中又用了FutureTask, 这20个线程是在等待它下一层的FutureTask执行完成。那么，为什么它们的下一层没有执行完成呢？
* 通过HomeApiControoler.java可以看到，代码中确实存在FutureTask内调FutureTask的情况，并且发现该类中所有的FutureTask（一共19种FutureTask）都是共用上面定义的threadPoolExecutor来执行的。
* 此时一种担忧油然而生：FutureTask内调FutureTask，并且都由同一个threadPoolExecutor来完成，靠谱吗？会不会存在类似死锁的问题？
* 在做实验之前答案并不确定，要看ThreadPoolExecutor是否能实现的这么好了。所以先看了下[ThreadPoolExecutor的java doc](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ThreadPoolExecutor.html)，并注意其中关于类似死锁相关逻辑的说明，果然，可以看到关于direct handoff的说明：

	    There are three general strategies for queuing:
	        Direct handoffs. A good default choice for a work queue is a SynchronousQueue that hands off tasks to threads without otherwise holding them. Here, an attempt to queue a task will fail if no threads are immediately available to run it, so a new thread will be constructed. This policy avoids lockups(死锁) when handling sets of requests that might have internal dependencies. Direct handoffs generally require unbounded maximumPoolSizes to avoid rejection of new submitted tasks. This in turn admits the possibility of unbounded thread growth when commands continue to arrive on average faster than they can be processed.
	        
* 从中可以知道用direct handoffs的方法可以避免线程池因为内部纯程间的依赖而造成的死锁
* 至此，答案已经较为明了，简言之：**业务线程在占用了线程池内所有的资源后又向线程池提交了新的任务，并且要等这些任务完成后才释放资源，而这些新提交的任务根本就没机会被完成！！！**
* 好了，我们来验证一下是否确实会这样：

	
		...
		   protected static final ThreadPoolExecutor executor = new ThreadPoolExecutor(2, 100, 2, TimeUnit.MINUTES, new LinkedBlockingQueue<Runnable>(),
		            new ThreadFactory() {
		                private AtomicInteger id = new AtomicInteger(0);

		                @Override
		                public Thread newThread(Runnable r) {
		                    Thread thread = new Thread(r);
		                    thread.setName("home-service-" + id.addAndGet(1));
		                    return thread;
		                }
		            }, new ThreadPoolExecutor.CallerRunsPolicy());

		   public static void main(String[] args) throws InterruptedException, ExecutionException {
		        Future<Long> f1 = executor.submit(new Callable<Long>() {
		            @Override
		            public Long call() throws Exception {
		                Thread.sleep(1000); //延时以使得第二层的f3在第一层的f2占用corePoolSize后才submit
		                Future<Long> f3 = executor.submit(new Callable<Long>() {
		                    @Override
		                    public Long call() throws Exception {

		                        return -1L;
		                    }
		                });
		                System.out.println("f1.f3" + f3.get());
		                return -1L;
		            }
		        });
		        Future<Long> f2 = executor.submit(new Callable<Long>() {
		            @Override
		            public Long call() throws Exception {
		                Thread.sleep(1000);//延时
		                Future<Long> f4 = executor.submit(new Callable<Long>() {
		                    @Override
		                    public Long call() throws Exception {

		                        return -1L;
		                    }
		                });
		                System.out.println("f2.f4" + f4.get());
		                return -1L;
		            }
		        });
		        System.out.println("here");
		        System.out.println("f1" + f1.get());
		        System.out.println("f2" + f2.get());
		    }
		...

运行程序后发现只打出一行日志"here", 并且进程没有结束，取其jstack：

		
		...
		"home-service-2" prio=5 tid=7fe2aa15b800 nid=0x10dcef000 waiting on condition [10dcee000]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <7f371ac80> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService$3.call(HomeService.java:136)
			at com.xxxxxx.productfront.service.HomeService$3.call(HomeService.java:1)
			at java.util.concurrent.FutureTask$Sync.innerRun(FutureTask.java:303)
			at java.util.concurrent.FutureTask.run(FutureTask.java:138)
			at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:895)
			at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:918)
			at java.lang.Thread.run(Thread.java:695)

		"home-service-1" prio=5 tid=7fe2aa0b8800 nid=0x10dbec000 waiting on condition [10dbeb000]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <7f36c5800> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService$2.call(HomeService.java:121)
			at com.xxxxxx.productfront.service.HomeService$2.call(HomeService.java:1)
			at java.util.concurrent.FutureTask$Sync.innerRun(FutureTask.java:303)
			at java.util.concurrent.FutureTask.run(FutureTask.java:138)
			at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:895)
			at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:918)
			at java.lang.Thread.run(Thread.java:695)
		...
		"main" prio=5 tid=7fe2ab000800 nid=0x1051c9000 waiting on condition [1051c8000]
		   java.lang.Thread.State: WAITING (parking)
			at sun.misc.Unsafe.park(Native Method)
			- parking to wait for  <7f369cab0> (a java.util.concurrent.FutureTask$Sync)
			at java.util.concurrent.locks.LockSupport.park(LockSupport.java:156)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:811)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:969)
			at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1281)
			at java.util.concurrent.FutureTask$Sync.innerGet(FutureTask.java:218)
			at java.util.concurrent.FutureTask.get(FutureTask.java:83)
			at com.xxxxxx.productfront.service.HomeService.main(HomeService.java:141)
		...

可见确实会造成'''死锁'''，而如果去掉f1和f2内的sleep延时，而把延时放到f2之前，则f1, f2, f3, f4均能成功完成。

## 结论
* 因为threadPoolExecutor使用了内部依赖，使得业务线程在占用了线程池内所有的资源后又向线程池提交了新的任务，并且要等这些任务完成后才释放资源，而这些新提交的任务根本就没机会被完成，从而造成业务线程没法完成，进而导致web服务器的业务线程耗尽，不再接受新的业务请求，即整个服务不可用。

## 如何处理
* 解决死锁：
  * 方法一：去掉threadPoolExecutor的内部依赖，每一层的future用各种的threadPoolExecutor，但现在各层层次较乱，即使现在理清了也很容易在之后被勿用。
  * 方法二：将maxPoolSize设为maximumPoolSize —— 不建议，耗用资源太多
  * 方法三：将queue的长度设为最小（貌似不能设为0，于是将其设为1），这样maxPoolSize就会生效，同时设置ThreadPoolExecutor.CallerRunsPolicy()会使得任务被pool拒绝时转由当前线程本地执行。
* 另外，初始化ThreadPoolExecutor时用的ThreadPoolExecutor.CallerRunsPolicy()也是没意义的，见java doc:

		Rejected tasks
	    New tasks submitted in method execute(java.lang.Runnable) will be rejected when the Executor has been shut down, and also when the Executor uses finite bounds for both maximum threads and work queue capacity, and is saturated. In either case, the execute method invokes the RejectedExecutionHandler.rejectedExecution(java.lang.Runnable, java.util.concurrent.ThreadPoolExecutor) method of its RejectedExecutionHandler. Four predefined handler policies are provided:
	        * In the default ThreadPoolExecutor.AbortPolicy, the handler throws a runtime RejectedExecutionException upon rejection.
	        * In ThreadPoolExecutor.CallerRunsPolicy, the thread that invokes execute itself runs the task. This provides a simple feedback control mechanism that will slow down the rate that new tasks are submitted.
	        * In ThreadPoolExecutor.DiscardPolicy, a task that cannot be executed is simply dropped.
	        * In ThreadPoolExecutor.DiscardOldestPolicy, if the executor is not shut down, the task at the head of the work queue is dropped, and then execution is retried (which can fail again, causing this to be repeated.)
	    It is possible to define and use other kinds of RejectedExecutionHandler classes. Doing so requires some care especially when policies are designed to work only under particular capacity or queuing policies.
	    
* 综合考虑，较好的使用方式是保持internal dependency并且如下设置ThreadPoolExecutor：

		ThreadPoolExecutor(20, 100, keepAliveTime, unit, new LinkedBlockingQueue<Runnable>(1), threadFactory)


## TODO
 * 平常并没有该死锁现象，为什么10月2号凌晨会出现呢？猜测是因为依赖服务(如缓存)突然变慢而使得第二层FutureTask的submit变慢导致的，需要进一步查明。

## 参考资料
 * [ThreadPoolExecutor的java doc](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ThreadPoolExecutor.html)
