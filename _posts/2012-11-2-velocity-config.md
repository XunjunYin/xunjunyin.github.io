---
layout: post
title: Velocity Config - Velocity配置优化
---

## 现象

* 性能测试，发现xx网站首页web接口的响应时间变慢，服务吞吐上不去
* 压力测试，并发数加大到20左右之后，系统的吞吐就不再增加，进程的cpu占用超过90%

## 观察和定位

* 使用50并发，不控制吞吐，对服务打压，取jstack
* 使用[stackAnalysis](https://github.com/xunjunyin/stackAnalysis)工具对取出的jstack进行处理后，可以看到堆栈内的很多线程都blocked在velocity相关的逻辑上

        "resin-tcp-connection-*:1907-57" daemon prio=10 tid=0x00002aaab8b09800 nid=0x522d waiting for monitor entry [0x0000000049779000]
           java.lang.Thread.State: BLOCKED (on object monitor)
                at java.util.Hashtable.get(Hashtable.java:333)
                - waiting to lock [***] (a org.apache.commons.collections.ExtendedProperties)
                at org.apache.commons.collections.ExtendedProperties.getBoolean(ExtendedProperties.java:1172)
                at org.apache.commons.collections.ExtendedProperties.getBoolean(ExtendedProperties.java:1157)
                at org.apache.velocity.runtime.RuntimeInstance.getBoolean(RuntimeInstance.java:1822)
                at org.apache.velocity.runtime.parser.node.ASTReference.init(ASTReference.java:139)
                at org.apache.velocity.runtime.parser.node.SimpleNode.init(SimpleNode.java:309)
                at org.apache.velocity.runtime.parser.node.SimpleNode.init(SimpleNode.java:309)
                at org.apache.velocity.runtime.parser.node.SimpleNode.init(SimpleNode.java:309)
                at org.apache.velocity.runtime.parser.node.SimpleNode.init(SimpleNode.java:309)
                at org.apache.velocity.runtime.parser.node.SimpleNode.init(SimpleNode.java:309)
                at org.apache.velocity.Template.initDocument(Template.java:227)
                at org.apache.velocity.Template.process(Template.java:135)
                at org.apache.velocity.runtime.resource.ResourceManagerImpl.loadResource(ResourceManagerImpl.java:437)
                at org.apache.velocity.runtime.resource.ResourceManagerImpl.getResource(ResourceManagerImpl.java:352)
                at org.apache.velocity.runtime.RuntimeInstance.getTemplate(RuntimeInstance.java:1533)
                at org.apache.velocity.runtime.directive.Parse.render(Parse.java:197)
                at org.apache.velocity.runtime.parser.node.ASTDirective.render(ASTDirective.java:207)
                at org.apache.velocity.runtime.parser.node.ASTBlock.render(ASTBlock.java:72)
                at org.apache.velocity.runtime.directive.Foreach.render(Foreach.java:420)
                at org.apache.velocity.runtime.parser.node.ASTDirective.render(ASTDirective.java:207)
                at org.apache.velocity.runtime.parser.node.ASTBlock.render(ASTBlock.java:72)
                at org.apache.velocity.runtime.parser.node.SimpleNode.render(SimpleNode.java:342)
                at org.apache.velocity.runtime.parser.node.ASTIfStatement.render(ASTIfStatement.java:106)
                at org.apache.velocity.runtime.parser.node.SimpleNode.render(SimpleNode.java:342)
                at org.apache.velocity.runtime.directive.Parse.render(Parse.java:260)
                at org.apache.velocity.runtime.parser.node.ASTDirective.render(ASTDirective.java:207)
                at org.apache.velocity.runtime.parser.node.ASTBlock.render(ASTBlock.java:72)
                at org.apache.velocity.runtime.directive.VelocimacroProxy.render(VelocimacroProxy.java:216)
                at org.apache.velocity.runtime.directive.RuntimeMacro.render(RuntimeMacro.java:311)
                at org.apache.velocity.runtime.directive.RuntimeMacro.render(RuntimeMacro.java:230)
                at org.apache.velocity.runtime.parser.node.ASTDirective.render(ASTDirective.java:207)
                at org.apache.velocity.runtime.parser.node.ASTBlock.render(ASTBlock.java:72)
                at org.apache.velocity.runtime.parser.node.ASTIfStatement.render(ASTIfStatement.java:87)
                at org.apache.velocity.runtime.parser.node.ASTBlock.render(ASTBlock.java:72)
                at org.apache.velocity.runtime.parser.node.ASTIfStatement.render(ASTIfStatement.java:87)
                at org.apache.velocity.runtime.parser.node.ASTBlock.render(ASTBlock.java:72)
                at org.apache.velocity.runtime.directive.Foreach.render(Foreach.java:420)
                at org.apache.velocity.runtime.parser.node.ASTDirective.render(ASTDirective.java:207)
                at org.apache.velocity.runtime.parser.node.SimpleNode.render(SimpleNode.java:342)
                at org.apache.velocity.Template.merge(Template.java:356)
                at org.apache.velocity.Template.merge(Template.java:260)
                at org.springframework.web.servlet.view.velocity.VelocityView.mergeTemplate(VelocityView.java:517)
                at org.springframework.web.servlet.view.velocity.VelocityLayoutView.doRender(VelocityLayoutView.java:167)
                at org.springframework.web.servlet.view.velocity.VelocityView.renderMergedTemplateModel(VelocityView.java:291)
                at org.springframework.web.servlet.view.AbstractTemplateView.renderMergedOutputModel(AbstractTemplateView.java:167)
                at org.springframework.web.servlet.view.AbstractView.render(AbstractView.java:250)
                at org.springframework.web.servlet.DispatcherServlet.render(DispatcherServlet.java:1063)
                at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:801)
                at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:719)
                at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:644)
                at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:549)
                at javax.servlet.http.HttpServlet.service(HttpServlet.java:115)
                at javax.servlet.http.HttpServlet.service(HttpServlet.java:92)
                at com.caucho.server.dispatch.ServletFilterChain.doFilter(ServletFilterChain.java:106)
                at com.caucho.filters.RewriteFilter.doFilter(RewriteFilter.java:120)
                at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:70)
                at outfox.cps.auth.filter.AuthFilter.chainFilter(AuthFilter.java:201)
                at outfox.cps.auth.filter.AuthFilter.doFilter(AuthFilter.java:83)
                at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:70)
                at account.app.CheckFilter.doFilter(CheckFilter.java:329)
                at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:70)
                at outfox.cps.web.filters.ArmaniRequestFilter.doFilter(ArmaniRequestFilter.java:115)
                at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:70)
                at toolbox.acs.client.Dog.doFilter(Dog.java:78)
                at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:70)
                at toolbox.flagfilter.FlagFilter.doFilter(FlagFilter.java:40)
                at com.caucho.server.dispatch.FilterFilterChain.doFilter(FilterFilterChain.java:70)
                at com.caucho.server.webapp.WebAppFilterChain.doFilter(WebAppFilterChain.java:173)
                at com.caucho.server.dispatch.ServletInvocation.service(ServletInvocation.java:229)
                at com.caucho.server.http.HttpRequest.handleRequest(HttpRequest.java:274)
                at com.caucho.server.port.TcpConnection.run(TcpConnection.java:511)
                - locked [***] (a java.lang.Object)
                at com.caucho.util.ThreadPool.runTasks(ThreadPool.java:516)
                at com.caucho.util.ThreadPool.run(ThreadPool.java:442)
                at java.lang.Thread.run(Thread.java:662)

* 基本可以认为是velocity的配置有问题，src/web/WEB-INF/cps-servlet.xml中velocity相关的配置如下：

        <bean id="velocityConfig"
            class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
            <property name="resourceLoaderPath" value="/WEB-INF/velocity/,/WEB-INF/vm/" />
            <property name="velocityPropertiesMap">
                <props>
                    <prop key="input.encoding">UTF-8</prop>
                    <prop key="output.encoding">UTF-8</prop>
                    <prop key="directive.set.null.allowed">true</prop>
                    <prop key="velocimacro.library">macro/cps_web_combo.vm,macro/cps_web_global.vm,macro/cps_gs.vm</prop>
                </props>
            </property>
        </bean>
        
* velocity的velocimacro.library.autoreload与file.resource.loader.cache默认值均为false, 若不进行配置, 则每次请求都会reload一次相关的vm文件，从而严重影响性能, 因此建议开发将配置改为如下：

        <bean id="velocityConfig"
            class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
            <property name="resourceLoaderPath" value="/WEB-INF/velocity/,/WEB-INF/vm/" />
            <property name="velocityPropertiesMap">
                <props>
                    <prop key="input.encoding">UTF-8</prop>
                    <prop key="output.encoding">UTF-8</prop>
                    <prop key="directive.set.null.allowed">true</prop>
                    <prop key="velocimacro.library.autoreload">false</prop>  //可不配置, 默认即为false
                    <prop key="velocimacro.context.localscope">true</prop>
                    <prop key="file.resource.loader.cache">true</prop>         //打开cache开关
                    <prop key="file.resource.loader.modificationCheckInterval">60</prop>    //load的间隔时间：其实若无动态修改的需求, 此处可改为-1，即只在启动时load一次, 此后不再load
                    <prop key="resource.manager.defaultcache.size">0</prop>      //resource.manager.defaultcache.size=0表示不限制cache大小
                    <prop key="velocimacro.library">macro/cps_web_combo.vm,macro/cps_web_global.vm,macro/cps_gs.vm</prop>
                </props>
            </property>
        </bean>

## 验证

* 修改配置后，性能大幅提升,由原来的90qps轻松飚至近300qps

## 经验

* 未设置file.resource.loader.cache=true时，对用户请求的每次响应都将reload一次vm资源(这与velocimacro.library.autoreload=true的效果相同)，从而严重影响性能
* 在性能测试时, 为寻找系统瓶颈, 可以对服务进行高并发打压，再取到stack进行分析(可借用工具/home/lisn/summaryJstack.pl)

## 参考文献

* [Velocity Developer Guide](http://velocity.apache.org/engine/releases/velocity-1.5/developer-guide.html)