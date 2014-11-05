---
layout: post
title: RPC client OOM - RPC client 内存泄露
---

## 原因简述

* 公司的rpc框架是内部开发并维护的
* RPC server timeout时不回复机制使得client端大量请求对象一直存活而不被销毁（内存泄露），造成client端内存耗尽

## 现象及分析过程

* 7.25(周四)晚某web卡死，发现是gc频繁所致，运维重启后服务正常
* 7.26(周五)对该web增加gc监控，对gc.time.max，gc.tenuredUsed.percent, gc.throughput做监控
* 7.29(周一)发现tenuredUsed.percent从上周五的40%升至50%. 找运维取jmap并用mat进行分析, 发现一个可疑的约900m的大对象odis.rpc2.RpcClient$ConnectionImpl(old区约4.5g)，其中积存了大量的Call对象（运行三天多后约16*1000=1.6万个）. 该RpcClient为向price server请求的RpcClient
* 查看odis.rpc2.RpcClient$ConnectionImpl相关code，分析pendingCalls进行put和remove的相关逻辑, 可知：client端的call对象们在完成send后会被put进pendingCalls里wait. 再通过一个后台线程向server读取各个call的结果, 该线程读到某个call的返回时就将该call从pendingCalls里remove

		odis.rpc2.RpcClient$ConnectionImpl部分代码: 

        @Override
        public void sendParam(Call call) throws RpcException {
            boolean error = true;
            try {
                pendingCalls.put(call.id, call);                    // yinxj: 将call请求放入map中
                // check closed after put to pendingCalls. if we check closed first, 
                // we may: 
                // 1. we check closed, it is false.
                // 2. connection is closed, all pendingCalls are discard.
                // 3. we put our call in pendingCalls and wait.
                // 4. we wait forever if we do not have a timeout.
                if (closed) {
                    throw new RpcException("connection already closed");
                }
        ...

        @Override
        public void run() {
            try {
                while (running) {
                    lastActiveTime = System.currentTimeMillis();
                    int id;
                    try {
                        id = in.readInt();
                    } catch (SocketTimeoutException e) {
                        continue;
                    }
                    Call call = pendingCalls.remove(id);                    // yinxj: 将对应的call从map中remove. 注意：此为唯一进行remove的逻辑, 若读不到call.id则该call将不会被remove
                    if (call == null) {
                        LOG.severe("Unknown call ID received: " + id
                                + ". This is usually because readFields and "
                                + "writeFields is not match, or RpcServer and "
                                + "RpcClient has different writable implements");
                        continue;
                    }
        ...

* jmap中odis.rpc2.RpcClient$ConnectionImpl中有众多的Call对象是因为其未被remove所致, 进一步可推断出是server端没有将call的请求回写导致call对象未从pendingCalls里remove. 与rpc相应开发人员确认: rpc server在处理某个请求时如果发现超时，就会直接结束该请求而不回复, 因此导致了pendingCalls的内存泄漏

## 解决办法

* 建议增加一后台线程去遍历pendingCalls，移除已timeout的无用calls -- 与rpc开发人员确认，会提一个issue进行fix
* 提升的rpc client的性能(超时情况较严重)

## QA

        Q: 日志里向rpc server请求的timeout数量每天约有4-5万，服务运行近4天，约为10-20w，但对应pendingCalls里的数量只有1.6w，二者的不一致性如何解释？
        A: pendingCalls中未remove的call仅为rpc server发现超时后未回复的请求, server在回写过程中才超时的call是会被remove掉的, 所以pendingCalls中的数量要小于总timeout的数量

        Q: 为何这个问题之前一直没有被发现？
        A: 原因大概有三个方面. 一是其他rpc server的timeout比例应该没有这么高, 因此pendingCalls里未被remove的对象不会太多; 二是我们的call对象较大(其中一个参数为500个url的数组), 参数占用空间较小的call对象造成的内存开销较小不易被发现; 三是我们的服务运行了一个多月后才内存耗尽的, 而之前上线较为频繁, 很少有运行这么长的时间

        Q: 测试过程中为什么也没发现这个问题？
        A: 我们曾模拟一些异常环境（如rpc server卡死不响应请求)做测试，但此bug复现的环境则需要rpc server所在的机器有较大负载(忙不过来而超时较多)，但亦能有一定的工作能力(能接收请求)而不是直接卡死(无法接收请求)，这样的环境不是很好模拟。我们之后会多考虑这样的风险

## 经验教训

* 运维：服务卡死时 如果知道是内存耗尽所致 最好取一下jmap再重启服务：jmap -dump:file=mem.dump <pid>, 以便更快的定位问题
* 测试：压力测试时需要深入了解各服务的实现原理，考虑并设计不同的异常环境进行测试，这样才能更好的规避风险
* 测试：在进行压力测试一段时间后，可以用jmap来取得内存dump并用mat来分析其中大小 top N 的对象, 排查异常的情况
