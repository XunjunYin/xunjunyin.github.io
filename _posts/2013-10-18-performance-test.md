---
layout: post
title: performance test rules & tools - 性能测试相关工具及规范
---

## 压力测试工具 
 * [jmeter](http://jmeter.apache.org/)
 * [Gatling] (http://gatling.io/)

## 测试指标
 * 吞吐(qps)
 * .avg响应时间及.99(.999)响应时间
 * 失败率
 * 超时率
 * 性能瓶颈(using stack analysis)
 * 系统资源占用(cpu, mem, 线程数, io, net, lsof, netstat, etc.)
 * 数据下载时间
 * 服务启动时间

### 指标确定
 * 新增接口：pv预估with pm, 一般要求.99在1s内
 * 改动接口: 一般不应差于线上

### 测试内容
 * 服务启动时间测试
   * 服务从启动到可正常服务的时间(多少分钟多少秒)
 * 接口测试
 * 负载测试(load testing)
   * 在不同负载下的性能情况, 判断能否达到预期指标
 * 系统瓶颈测试
   * 在服务性能范围之外高并发打压测试服务, 取stack, 分析系统瓶颈, 并判断是否可改进
 * 最大性能测试(stress testing)
   * 给出瓶颈项在一定指标下, 系统的最大性能，以及压力增大时可能的解决方案（如瓶颈为cpu, 则可给出load在小于cpu核数时的性能情况, 解决方案即为换用cpu更强【核数多或处理能力强】的机器或负载均衡）
 * 依赖项测试（假设A依赖B）
   * 强弱依赖关系测试
   * B异常(超时, 接口exception等异常)时A是否能合理处理
   * B异常时打压A, A的性能(线程数、连接数)是否合理, 原则: 尽量少的影响用户功能, 尽大可能的保证服务内部正常
 * 异常测试
   * 可用access log作为query进行该测试
   * 对服务进行打压, 找出日志中所有的exception
   * 统计, 分析各种exception是否正常, 与开发沟通
   * 分析测试服务access log中服务器状态异常的用户请求, 与开发沟通, 并应尽量避免500, 503等错误(老接口可用线上access log作为query进行该测试)
 * 稳定性测试
   * 模拟线上情况(各接口的比例)
     * cpu占用及load
     * gc情况
     * 线程数
     * io
     * net
     * 判断是否有定时任务的影响
   * 内存使用测试
     * 取jmap -dump:file=mem.dump <pid>, 再用mat分析其中top10（或topN)的对象，判断其数量及大小是否合理

### 检查
 * db索引检查
 * 连接池线程池的检查
 * 代码严重静态bug检查, husdon + findbugs
 * 代码质量：单元测试通过率及覆盖率、代码重复率、注释率等

### 注意事项
 * 不对线上服务打压力(注意依赖关系)
 * 测试环境与线上环境的一致性
   * 物理环境
   * 数据量(mysql, index, etc.)
   * 熟悉代码及功能，保证代码分支的覆盖程度

### 工具
 * vmstat
   * vmstat reports information about processes, memory, paging, block IO, traps, and cpu activity.
 * jstack
   * 分析java进程各线程的状态
 * [stackAnalysis](https://github.com/xunjunyin/stackAnalysis)
  * 未发布到github, 敬请期待
 * pstack
   * 分析进程各线程状态，对任何pid均可使用
 * strace
   * 查看分析系统调用
 * jstat
   * java进程统计性的监控工具（包括gc情况）
 * jconsole
   * 观察java进程内存、cpu等的详细使用情况
 * visualvm
   * 更NB的jconsole+jstat
 * [stress](http://manpages.ubuntu.com/manpages/lucid/man1/stress.1.html)
   * 对机器的虚拟压力使用工具(如占用多少内存多少cpu多长时间）
   * 见love39: /disk1/xjyin/tools/
 * [gcviewer](https://github.com/chewiebug/GCViewer)
   * 分析gclog
 * jmap
   * 分析java进程内存
 * jhat
   * 分析java进程内存: 会有详细的结果
 * [mat](http://www.eclipse.org/mat/)若在机群上可以配合vnc使用
   * 更NB的jhat, 一般用于内存泄露等对内存较为详细的分析