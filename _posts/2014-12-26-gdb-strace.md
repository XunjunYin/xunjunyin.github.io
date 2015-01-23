---
layout: post
title: gdb & strace追踪jdk bug
---

### 现象
* java应用的web服务器突然挂掉，无任何jvm相关日志，重启后不久再次挂掉
* 再次重启，不久后机器挂掉【机器为虚拟机】

### 相关日志
* 到宿主机器重启机器后，查看dmesg，可看到有相关信息：

		[46019.223344] 3065881 pages non-shared
		[46019.223348] Out of memory: kill process 16211 (java) score 1135790 or a child
		[46019.225305] Killed process 16211 (java)
		[46019.293729] java invoked oom-killer: gfp_mask=0x201da, order=0, oom_adj=0
		[46019.293734] java cpuset=/ mems_allowed=0
		[46019.293750] Pid: 2187, comm: java Not tainted 2.6.32-5-amd64 #1
		[46019.293752] Call Trace:
		[46019.293761]  [<ffffffff810b643c>] ? oom_kill_process+0x7f/0x23f
		[46019.293765]  [<ffffffff8106bb5e>] ? timekeeping_get_ns+0xe/0x2e
		[46019.293768]  [<ffffffff810b6960>] ? __out_of_memory+0x12a/0x141
		[46019.293771]  [<ffffffff810b6ab7>] ? out_of_memory+0x140/0x172
		[46019.293775]  [<ffffffff810ba81c>] ? __alloc_pages_nodemask+0x4ec/0x5fc
		[46019.293780]  [<ffffffff810bbd85>] ? __do_page_cache_readahead+0x9b/0x1b4
		[46019.293784]  [<ffffffff810bbeba>] ? ra_submit+0x1c/0x20
		[46019.293787]  [<ffffffff810b4b87>] ? filemap_fault+0x17d/0x2f6
		[46019.293793]  [<ffffffff810cab26>] ? __do_fault+0x54/0x3c3
		[46019.293796]  [<ffffffff810cce7a>] ? handle_mm_fault+0x3b8/0x80f
		[46019.293801]  [<ffffffff812ff306>] ? do_page_fault+0x2e0/0x2fc
		[46019.293805]  [<ffffffff812fd1a5>] ? page_fault+0x25/0x30

 * 说明进程占用内存过高，被系统的oom_killer强行kill所致
 * 猜测机器会挂的原因是：该进程占用内存过快过高，oom_killer来不及动作便已惨遭殃及
 
### 初步分析及方案拟定
* 尝试再次复现，用jdb attach 到java进程进行remote debug，【久经波折后】发现某个分页请求数据的接口会间歇性的触发该现象，用vmstat观察机器内存使用，发现发生问题时内存下降异常迅速，约每秒100M，数十秒内就会吃尽机器所有内存
* 该java进程Xmx配置为6G，且独占该机器。机器的内存有11G
* 结合之前的经验，进程占用内存远超Xmx的现象，初步认为很可能是jni所致。将工程中所有最新用到jni的地方review，未能找到明确线索。
* 在内存急剧下降期间对java进程取mem dump和jstack都未能看到异常现象
* 考虑到内存消耗的速度，决定用strace来看故障期间的系统调用，命令为：

        strace -f -t -T -e trace=all -p 20390 2<&1 | tee -a 20390.strace.log

* 故障发生时，需要尽快杀掉进程，以避免殃及系统挂掉。而gdb正好可以将进程suspend，并同时观察线程堆栈，可以用来辅助分析
* 尝试后发现strace和gdb不能同时attach到进程，于是初步将分析方案定为：
  * 出现故障时，先用strace打出系统调用，打十秒左右
  * 停掉strace，用gdb attach到进程，使进程挂起，一方面阻止内存的消耗，另一方面可用于分析 

### 复现及分析
* 如上所述方案，得到故障期间系统调用异常的地方为：

		[pid 21832] 17:15:26 clock_gettime(CLOCK_MONOTONIC,  <unfinished ...>
		[pid 21751] 17:15:26 futex(0x42564324, FUTEX_WAIT_BITSET_PRIVATE|FUTEX_CLOCK_REALTIME, 1, {1419585326, 152497000}, ffffffff <unfinished ...>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1e5d000, 32768, PROT_READ|PROT_WRITE <unfinished ...>
		[pid 21832] 17:15:26 <... clock_gettime resumed> {19389, 378140695}) = 0 <0.000111>
		[pid 21747] 17:15:26 <... mprotect resumed> ) = 0 <0.000096>
		[pid 21832] 17:15:26 clock_gettime(CLOCK_MONOTONIC, {19389, 378329426}) = 0 <0.000045>
		[pid 21832] 17:15:26 clock_gettime(CLOCK_MONOTONIC, {19389, 378442545}) = 0 <0.000044>
		[pid 21832] 17:15:26 gettimeofday({1419585326, 103206}, NULL) = 0 <0.000046>
		[pid 21832] 17:15:26 futex(0x7ff9b878d1e4, FUTEX_WAIT_BITSET_PRIVATE|FUTEX_CLOCK_REALTIME, 1, {1419585326, 153206000}, ffffffff <unfinished ...>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1e65000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000060>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1e6d000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000045>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1e75000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000062>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1e7d000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000043>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1e85000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000056>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1e8d000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000104>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1e95000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000044>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1e9d000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000062>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1ea5000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000044>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1ead000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000055>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1eb5000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000057>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1ebd000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000045>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1ec5000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000045>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1ecd000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000043>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1ed5000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000044>
		[pid 21747] 17:15:26 mprotect(0x7ff8e1edd000, 32768, PROT_READ|PROT_WRITE) = 0 <0.000055>
		
* 可以发现mprotect方法调用频繁【结合故障出现之间的系统调用进行对比】，且全在21747的线程内，查看doc可知每次会malloc 32K的内存
* 用gdb suspend进程后，查看21747对应的线程，并执行dt得到其堆栈：

		Breakpoint 1, 0x00007ff9e12b14e0 in mprotect () from /lib/x86_64-linux-gnu/libc.so.6
		(gdb) bt
		#0  0x00007ff9e12b14e0 in mprotect () from /lib/x86_64-linux-gnu/libc.so.6
		#1  0x00007ff9e1254671 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
		#2  0x00007ff9e1255b90 in malloc () from /lib/x86_64-linux-gnu/libc.so.6
		#3  0x00007ff9e0cd83f8 in os::malloc(unsigned long) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#4  0x00007ff9e07f5f8c in ChunkPool::allocate(unsigned long) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#5  0x00007ff9e07f572a in Chunk::operator new(unsigned long, unsigned long) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#6  0x00007ff9e07f5d11 in Arena::grow(unsigned long) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#7  0x00007ff9e0cbffa8 in Node::out_grow(unsigned int) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#8  0x00007ff9e07c63e1 in Node::add_out(Node*) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#9  0x00007ff9e0c4b785 in PhaseIdealLoop::clone_loop(IdealLoopTree*, Node_List&, int, Node*) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#10 0x00007ff9e0c4fa59 in PhaseIdealLoop::partial_peel(IdealLoopTree*, Node_List&) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#11 0x00007ff9e0c32dac in IdealLoopTree::iteration_split_impl(PhaseIdealLoop*, Node_List&) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#12 0x00007ff9e0c33000 in IdealLoopTree::iteration_split(PhaseIdealLoop*, Node_List&) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#13 0x00007ff9e0c32f68 in IdealLoopTree::iteration_split(PhaseIdealLoop*, Node_List&) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#14 0x00007ff9e0c41095 in PhaseIdealLoop::build_and_optimize(bool, bool) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#15 0x00007ff9e097134f in Compile::Optimize() () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#16 0x00007ff9e096de84 in Compile::Compile(ciEnv*, C2Compiler*, ciMethod*, int, bool, bool) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#17 0x00007ff9e08f0d3e in C2Compiler::compile_method(ciEnv*, ciMethod*, int) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#18 0x00007ff9e09786aa in CompileBroker::invoke_compiler_on_method(CompileTask*) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#19 0x00007ff9e0977f95 in CompileBroker::compiler_thread_loop() () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#20 0x00007ff9e0df0539 in compiler_thread_entry(JavaThread*, Thread*) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#21 0x00007ff9e0de9a41 in JavaThread::run() () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#22 0x00007ff9e0ce0d1f in java_start(Thread*) () from /global/install/jdk1.6.0_35/jre/lib/amd64/server/libjvm.so
		#23 0x00007ff9e176eb50 in start_thread () from /lib/x86_64-linux-gnu/libpthread.so.0
		#24 0x00007ff9e12b4a7d in clone () from /lib/x86_64-linux-gnu/libc.so.6
		#25 0x0000000000000000 in ?? ()
		(gdb) cont
		Continuing.
		
* 由各frame中各方法名：CompileBroker::compiler_thread_loop(), Compile::Optimize()等，可以推断出此时应该与jvm类编译优化相关逻辑有关。结合已有先验知识，知道jvm对类的编译会与调用次数等因素有关。
* 在mprotect处下断点再cont发现会继续进入该断点，多次cont依旧如此
* 至此基本可认定是jvm的bug，查看机器的jdk版本：

        java -version
        java version "1.6.0_35"
        Java(TM) SE Runtime Environment (build 1.6.0_35-b10)
        Java HotSpot(TM) 64-Bit Server VM (build 20.10-b01, mixed mode)

* 而1.6最新的为1.6.0_45, 因此先不深究jvm具体的bug所在，先做升级
* 升级是否修复该故障，且听下回分解:) 

### 后续
* 现在回头来分析该bug触发的逻辑，发现是测试同学为了便于测试，将该分页获取数据接口由每次获取20条改为了每次获取200条。
* 在web启动之后，若先用20的分页进地调用，则会让jvm“预热”的优化该编译优化逻辑，不会触发。而若在web启动之后立刻用200的分页请求，则必然会触发该bug
* 有必要对jdk源码不同版本进行对比以确认相关逻辑是否已经优化，当然不排队jdk还存在类似隐藏较深的bug

### 下回分解
* 很悲伤！！！升级jdk（1.6）后该bug依旧存在，说明该bug未被修复
* 决定尝试设置jni相关参数来试图绕过该bug, 于是设置-XX:CompileThreshold=1, 发现bug不再能复现。但同时发现web服务启动时间变长，原因应该是做了大量的编译工作，并且web启动之后，cpu占用波动较大，持续较长时间后才回归平稳————依旧是不断在做编译
* 另外，根据doc:

			-Xint, -Xcomp, and -Xmixed
		The two flags -Xint and -Xcomp are not too relevant for our everyday work, but highly interesting in order to learn something about the JVM. 
		The -Xint flag forces the JVM to execute all bytecode in interpreted mode, which comes along with a considerable slowdown, usually factor 10 or higher. 
		On the contrary, the flag -Xcomp forces exactly the opposite behavior, that is, the JVM compiles all bytecode into native code on first use, thereby applying maximum optimization level. 
		This sounds nice, because it completely avoids the slow interpreter. 
		However, many applications will also suffer at least a bit from the use of -Xcomp, even if the drop in performance is not comparable with the one resulting from -Xint. 
		The reason is that by setting-Xcomp we prevent the JVM from making use of its JIT compiler to full effect.
		The JIT compiler creates method usage profiles at run time and then optimizes single methods (or parts of them) step by step, and sometimes speculatively, to the actual application behavior. 
		Some of these optimization techniques, e.g., optimistic branch prediction, cannot be applied effectively without first profiling the application. 
		Another aspect is that methods are only getting compiled at all when they prove themselves relevant, i.e., constitute some kind of hot spot in the application. 
		Methods that are called rarely (or even only once) are continued to be executed in interpreted mode, thus saving the compilation and optimization cost.

* 可以知道上述配置或是直接用Xcomp虽然看似可以避免该bug的触发，但会对性能有较大的损伤，因为jit会根据调用次数及性能的统计信息来优化bytecode，如果直接comp就得不到这些统计信息优化的不够好了

### 处理方案
* 不做处理：考虑到之前线上服务表现平稳，从未触发该bug，以及该bug要么在服务启动之后尽快出现，要么不会出现，因此暂不做优化调整，而是上线后观察两三分钟，不触发该bug才认为上线成功
* 给oracle报bug
* 考虑升级jdk到1.7或1.8
 
### 相关阅读
* [JVM性能优化1-JVM简介](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=202029662&idx=1&sn=5fbf34c5f4636bbc6712afff94056f41#rd)
* [JVM性能优化2-编译器](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=202047314&idx=1&sn=92080ce6f14f58103cc81a5c452c68dc#rd)
* [JVM性能优化3-垃圾回收](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=202093140&idx=1&sn=280bb4e93f7e020d6cdfbbd55de0e3e5#rd)
* [JVM性能优化4-C4垃圾回收](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=202109974&idx=1&sn=c559222b5d7b9477f011feb15b14f797#rd)
* [JVM性能优化5-Java的伸缩性](http://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=202127705&idx=1&sn=bfb366d8af6c03f61e46664d28af7d3a#rd)
* [Useful JVM Flags 1-JVM Types and Compiler Modes](https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-1-jvm-types-and-compiler-modes/)
* [Useful JVM Flags 2-Flag Categories and JIT Compiler Diagnostics](https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-2-flag-categories-and-jit-compiler-diagnostics/)
* [Useful JVM Flags 3-Printing all XX Flags and their Values](https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-3-printing-all-xx-flags-and-their-values/)
* [Useful JVM Flags 4-Heap Tuning](https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-4-heap-tuning/)
* [Useful JVM Flags 5-Young Generation Garbage Collection](https://blog.codecentric.de/en/2012/08/useful-jvm-flags-part-5-young-generation-garbage-collection/)
* [Useful JVM Flags 6-Throughput Collector](https://blog.codecentric.de/en/2013/01/useful-jvm-flags-part-6-throughput-collector/)
* [Useful JVM Flags 7-CMS Collector](https://blog.codecentric.de/en/2013/10/useful-jvm-flags-part-7-cms-collector/)
* [Useful JVM Flags 8-GC Logging](https://blog.codecentric.de/en/2014/01/useful-jvm-flags-part-8-gc-logging/)

### 致谢
* 感谢[北京新观念技术服务有限公司](http://www.xinitek.com)CEO李斯宁提供技术支持与分析讨论
