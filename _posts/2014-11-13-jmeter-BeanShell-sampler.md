---
layout: post
title: BeanShell Sampler - Jmeter
---

## 用prop来存放对象供多线程共用
* 在用jmeter打压力时，由于是做rpc调用的请求，用beanshell sampler来发送请求，脚本如下

		import com.netease.love.timeline.ITimelineProtocol;
		import com.netease.tinyrpc.MultiNodeRpcProxyFactory;
		import com.netease.tinyrpc.locator.FixedModHashLocator;

		factory = new MultiNodeRpcProxyFactory();
		proxies = factory.createProxy("114.113.201.07:29011,114.113.201.09:29011,114.113.201.10:29011", ITimelineProtocol.class, FixedModHashLocator.class);
		try {
			//proxies.deleteDirectMessage(-1264136337218617121L, 9223219345691512173L, 1551505303962223348L, false);
			proxies.deleteDirectMessage(Long.parseLong(vars.get("userId")), Long.parseLong(vars.get("sessionId")), Long.parseLong(vars.get("messageId")), false);
		//	return vars.get("userId")+ vars.get("sessionId") + vars.get("messageId");
		} catch (Exception e) {
			System.out.println("error");	
			return "false";
		}
		System.out.println("done");	

		return "true";

* 用该脚本做调试请求，一切正常，不过有个很明显的问题：每个请求都会new一个proxy，费时费资源。尝试着用来跑了2000个请求，便OOM了。
* 解决方案：用单例来存放proxies对象，如下（非严格的单例）：

		import com.netease.love.timeline.ITimelineProtocol;
		import com.netease.tinyrpc.MultiNodeRpcProxyFactory;
		import com.netease.tinyrpc.locator.FixedModHashLocator;

		ITimelineProtocol proxies = props.get("proxies");
		if(proxies ==  null) {
			factory = new MultiNodeRpcProxyFactory();
			proxies = factory.createProxy("114.113.201.07:29011,114.113.201.09:29011,114.113.201.10:29011", ITimelineProtocol.class, FixedModHashLocator.class);
			props.put("proxies", proxies);
			
		}
		//System.out.println("null=" + (proxies == null));
		//System.out.println("begin");	
		//ITimelineProtocol proxies = factory.createProxy("114.113.201.07:29011,114.113.201.09:29011,114.113.201.10:29011", ITimelineProtocol.class, FixedModHashLocator.class);
		try {
			//-1264136337218617121	9223219345691512173	1551505303962223348
			//proxies.deleteDirectMessage(-1264136337218617121L, 9223219345691512173L, 1551505303962223348L, false);
			proxies.deleteDirectMessage(Long.parseLong(vars.get("userId")), Long.parseLong(vars.get("sessionId")), Long.parseLong(vars.get("messageId")), false);
		//	return vars.get("userId")+ vars.get("sessionId") + vars.get("messageId");
		} catch (Exception e) {
			System.out.println("error");	
			return "false";
		}
		System.out.println("done");	

		return "true";
		
* 再次打压，不再OOM。