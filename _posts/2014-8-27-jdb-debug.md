---
layout: post
title: java jdb remote debug
---


# jdb remote debug - java远程调试

## 简介 
 * 用log来做调试的方法低效茫目
 * 远程调试是jdk自带的一个有利调试工具，可以快速定位问题

## 在工作机上使用eclipse来自带的remote debug

### 问题
 * 运维禁止使用remote debug, 因为若网络边界被攻破，内网开启java debug的机器将面临着巨大的风险。

### 使用端口转发来实现eclipse远程调试
* 给服务器的java进程加入调试参数：
  * -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=127.0.0.1:8090,suspend=n 
  * 本文以8090为例
* 用ssh工具的端口转发功能将远程端口映射到本地端口：
  * 在xshell在tunneling(隧道)处设置
    * 类型为outgoing
    * 源主机及port为localhost:8090
    * 目标主机及port为127.0.0.1:8090
  * 在scureCRT中在port forwarding处设置
    * remote host为localhost:8090
    * local address为127.0.0.1:8090
  * 若是mac或linux系统，可以直接执行下面的命令来进行tcp的端口转发:
    * ssh -f -L 8092:localhost:8092 love@114.113.201.54 -p 16322 sleep 20
* 在eclipse中新建一个remote debug
  * host为localhost
  * port为8090
 

## 在服务器上使用jdb

### 背景
 * eclipse remote debug(封装了jdb)：奇慢无比，attach上往往需要几分钟。原因不明。。。应该是北京办公室和机房之间网络的问题

### 使用方法
* 开端口——给待调试的java进程增加启动参数：
  * -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=127.0.0.1:8090,suspend=n 
  * '''suspend=n/y指是否在进程启动时等待调试器attach, 调试一个非server的java进程时可以将suspend设为y，否则可能在attach上之前就已经执行完了'''
* 连接：jdb -connect com.sun.jdi.SocketAttach:hostname=localhost,port=8090
  * 我在机群上加了个alias为jdbp，直接执行jdbp 8090为同样的命令

### 帮助说明
 * help（所有用法都能通过help看到）

### 我用过的几个命令
* 使用源代码
  * use /disk1/love-audit/branch-2.2.5/src/main/java
* 下断点
  * stop at com.netease.love.TopicStore:19
* step控制
  * step                      -- execute current line
  * step up                   -- execute until the current method returns to   * its caller
  * stepi                     -- execute current instruction
  * next                      -- step one line (step OVER calls)
  * cont                      -- continue execution from breakpoint
* 查看变量
  * locals
* 打印变量
  * dump paramNames
  * dump paramNames.elementData
  * print paramNames
* 查看栈
  * where
* 查看所有线程
  * threads
* 查看源代码
  * list


### 实战示例（部分）
resin-port-18082-45[1] next

\> 
Step completed: "thread=resin-port-18082-45", com.netease.love.topic.store.TopicDbStore.getTopics(), line=98 bci=41
98                sql = parseStatement(Topic.class, "selectWithLimit");

resin-port-18082-45[1] list
94                sql = parseStatement(Topic.class, "selectByStateWithLimit");
95                namedParameters.put("state", Const.TOPIC_STATE.ONLINE);  
96            }
97            else {
98 =>             sql = parseStatement(Topic.class, "selectWithLimit");
99            }
100            namedParameters.put("offset", offset); 
101            namedParameters.put("length", length);
102            List<Topic> topics = namedParameterJdbcTemplate.query(sql, namedParameters, Topic.ROW_MAPPER);
103            return topics;
resin-port-18082-45[1] next

\> 
Step completed: "thread=resin-port-18082-45", com.netease.love.topic.store.TopicDbStore.getTopics(), line=100 bci=52
100            namedParameters.put("offset", offset); 

resin-port-18082-45[1] next

\> 
Step completed: "thread=resin-port-18082-45", com.netease.love.topic.store.TopicDbStore.getTopics(), line=101 bci=66
101            namedParameters.put("length", length);

resin-port-18082-45[1] locals
Method arguments:
publishedOnly = false
offset = 0
length = 20
Local variables:
namedParameters = instance of java.util.HashMap(id=11628)
sql = "SELECT id, type, publish_id, title, context, pub_time, edit_time, editor, discuss_count, state FROM Topic_Topic  ORDER BY publish_id DESC LIMIT :limit OFFSET :offset"
