---
layout: post
title: GenericObjectPool使用优化
---

### 背景

* 某应用1.0性能测试
* 服务强依赖于mysql, 许多接口都会请求mysql
* 对mysql的请求用GenericObjectPool的连接池来进行管理, 设置如下：
 
		 79             connectionPool.setMaxActive(maxActive);
		 80             connectionPool.setTestOnBorrow(false);
		 81             connectionPool.setTestOnReturn(true);
		 82             connectionPool.setTestWhileIdle(true);
		 83             connectionPool
		 84                 .setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
		 85                 
		 86             // Start the Evictor
		 87             connectionPool
		 88                 .setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);

### 问题

* 用100并发打压服务，发现拿到连接并在干活的线程数只有10+, 而其余80+的线程wait在borrowObject的逻辑, 相应的stack如下：

		有许多线程在blocked在makeObject相关的逻辑，均在等一把锁
		1462 "resin-tcp-connection-*:3231-321" daemon prio=10 tid=0x000000004dc43800 nid=0x65f5 waiting for monitor entry [0x00000000507ff000]
		1463    java.lang.Thread.State: BLOCKED (on object monitor)
		1464         at org.apache.commons.dbcp.PoolableConnectionFactory.makeObject(PoolableConnectionFactory.java:290)
		1465         - waiting to lock <0x00000000b26ee8a8> (a org.apache.commons.dbcp.PoolableConnectionFactory)
		1466         at org.apache.commons.pool.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:771)
		1467         at org.apache.commons.dbcp.PoolingDataSource.getConnection(PoolingDataSource.java:95)
		1468         at outfox.cps.dao.source.CpsDataSource.getConnection(CpsDataSource.java:61)
		1469         at org.springframework.jdbc.datasource.DataSourceUtils.doGetConnection(DataSourceUtils.java:111)
		1470         at org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:77)
		1471         at org.springframework.jdbc.core.JdbcTemplate.execute(JdbcTemplate.java:572)
		1472         at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:636)
		1473         at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:665)
		1474         at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:673)
		1475         at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:713)
		1476         at outfox.cps.dao.impl.FriendshipDaoImpl.getFansWithTime(FriendshipDaoImpl.java:93)
		......

		有一个线程拿到了makeObject相关的锁，并在执行相关的逻辑
		1208 "resin-tcp-connection-*:3231-149" daemon prio=10 tid=0x000000004d67e800 nid=0x66d7 runnable [0x000000005180f000]
		1209    java.lang.Thread.State: RUNNABLE
		1210         at java.net.SocketInputStream.socketRead0(Native Method)
		1211         at java.net.SocketInputStream.read(SocketInputStream.java:129)
		1212         at com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:113)
		1213         at com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:160)
		1214         at com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:188)
		1215         - locked <0x000000009f5a4cc8> (a com.mysql.jdbc.util.ReadAheadInputStream)
		1216         at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:2329)
		1217         at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:2774)
		1218         at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:2763)
		1219         at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3299)
		1220         at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:1837)
		1221         at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:1961)
		1222         at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2537)
		1223         - locked <0x000000009f5a1600> (a java.lang.Object)
		1224         at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2466)
		1225         at com.mysql.jdbc.StatementImpl.executeQuery(StatementImpl.java:1383)
		1226         - locked <0x000000009f5a1600> (a java.lang.Object)
		1227         at com.mysql.jdbc.ConnectionImpl.loadServerVariables(ConnectionImpl.java:3752)
		1228         at com.mysql.jdbc.ConnectionImpl.initializePropsFromServer(ConnectionImpl.java:3345)
		1229         at com.mysql.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:2046)
		1230         at com.mysql.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:729)
		1231         at com.mysql.jdbc.JDBC4Connection.<init>(JDBC4Connection.java:46)
		1232         at sun.reflect.GeneratedConstructorAccessor26.newInstance(Unknown Source)
		1233         at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:27)
		1234         at java.lang.reflect.Constructor.newInstance(Constructor.java:513)
		1235         at com.mysql.jdbc.Util.handleNewInstance(Util.java:406)
		1236         at com.mysql.jdbc.ConnectionImpl.getInstance(ConnectionImpl.java:302)
		1237         at com.mysql.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:283)
		1238         at java.sql.DriverManager.getConnection(DriverManager.java:582)
		1239         at java.sql.DriverManager.getConnection(DriverManager.java:207)
		1240         at org.apache.commons.dbcp.DriverManagerConnectionFactory.createConnection(DriverManagerConnectionFactory.java:46)
		1241         at org.apache.commons.dbcp.PoolableConnectionFactory.makeObject(PoolableConnectionFactory.java:290)
		1242         - locked <0x00000000b26ee8a8> (a org.apache.commons.dbcp.PoolableConnectionFactory)
		1243         at org.apache.commons.pool.impl.GenericObjectPool.borrowObject(GenericObjectPool.java:771)
		1244         at org.apache.commons.dbcp.PoolingDataSource.getConnection(PoolingDataSource.java:95)
		1245         at outfox.cps.dao.source.CpsDataSource.getConnection(CpsDataSource.java:61)
		1246         at org.springframework.jdbc.datasource.DataSourceUtils.doGetConnection(DataSourceUtils.java:111)
		1247         at org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:77)
		1248         at org.springframework.jdbc.core.JdbcTemplate.execute(JdbcTemplate.java:572)
		1249         at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:636)
		1250         at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:665)
		1251         at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:673)
		1252         at org.springframework.jdbc.core.JdbcTemplate.query(JdbcTemplate.java:713)
		1253         at outfox.cps.dao.impl.FriendshipDaoImpl.getAttentionsWithTime(FriendshipDaoImpl.java:69)
		1254         at outfox.cps.api.impl.FriendshipApi$RelationListOperation.copyRealtionListFromDbToRedis(FriendshipApi.java:118)
		1255         at outfox.cps.api.impl.FriendshipApi$RelationListOperation.getRelationList(FriendshipApi.java:144)
		......


### 分析
* 为什么maxActive设置为100, 还会有如此多的线程在makeObject?这是我们的setMaxActive没有生效, 还是GenericObjectPool有其他destroy对象的机制被我们忽略了?
* 查看GenericObjectPool的java doc, 了解对象池关于对象创建和销毁相关的方式以及配置方式
* 打压开始后, 动态debug, 发现pool对象有许多我们未进行配置的成员变量，对照doc逐一check各变量
发现一个可怀疑点: numIdle, 其值一直在2-8之间波动, 且doc中如是描述:
 
		maxActive controls the maximum number of objects that can be borrowed from the pool at one time. When non-positive, there is no limit to the number of objects that may be active at one time. When maxActive is exceeded, the pool is said to be exhausted. The default setting for this parameter is 8.
		maxIdle controls the maximum number of objects that can sit idle in the pool at any time. When negative, there is no limit to the number of objects that may be idle at one time. The default setting for this parameter is 8.
* 可知默认的maxIdle值为8，若未进行配置，当pool中idle的object超过默认值8时，多余的对象就会被destroy
* 至此，问题基本定位到

### 验证

* 增加对maxIdle的配置，即：
 
		 79             connectionPool.setMaxActive(maxActive);
		 80             connectionPool.setMaxIdle(maxActive);    //实际中可以设小一些, 如30
		 81             connectionPool.setTestOnBorrow(false);
		 82             connectionPool.setTestOnReturn(true);
		 83             connectionPool.setTestWhileIdle(true);
		 84             connectionPool
		 85                 .setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
		 86                 
		 87             // Start the Evictor
		 88             connectionPool
		 89                 .setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);
		 
* 重新打压, 发现吞吐增长近一倍，响应速度也显著加快，stack中无线程blocked在borrowObject或makeObject相关的逻辑

### 另一个重要的问题

* 未设置maxWait, doc中对maxWait说明如下：
 
		setMaxWait
		
		public void setMaxWait(long maxWait)
		Sets the maximum amount of time (in milliseconds) the borrowObject() method should block before throwing an exception when the pool is exhausted and the "when exhausted" action is WHEN_EXHAUSTED_BLOCK. When less than or equal to 0, the borrowObject() method may block indefinitely.
		Parameters:
		maxWait - maximum number of milliseconds to block when borrowing an object.
		See Also:
		getMaxWait(), setWhenExhaustedAction(byte), WHEN_EXHAUSTED_BLOCK
* 可知若不进行设置，当被依赖的服务由于某些故障（如机器资源被某进程大量占用）而响应极慢时，会有大量线程blocked在borrowObject的逻辑，最终导致resin线程耗尽，服务卡死，用户请求被拒绝
* 可参考之前的一次线上故障：某服务变慢导致依赖于它的另一web卡死，其原因为改web请求改服务时的连接池未对borrowObject设置timeout，导致大量线程blocked在borrowObject的逻辑
* 因此需要对maxWait进行设置，且设置的时间不宜过大(高吞吐时仍有可能导致web卡死)，也不宜过小（可能导致较多的请求失败），具体的配置可参考如下：
 
		基本原则：
		qt<N, 其中q为服务最大吞吐(请求/s), t为设置的maxWait时间(s), N为resin的最大线程数

		resin的线程数当前为512, 预估最大吞吐为100, 因此有t<N/q=512/100=5.12
		我们将其配置为3s(即3000ms), 从而当该对象池中的对象出现异常时，仍可扛住512/3约为170qps的压力
* 具体的配置如下：
 
		 79             connectionPool.setMaxActive(maxActive);
		 80             connectionPool.setMaxIdle(maxActive);
		 81             connectionPool.setMaxWait(maxWait);    // maxWait=3000
		 82             connectionPool.setTestOnBorrow(false);
		 83             connectionPool.setTestOnReturn(true);
		 84             connectionPool.setTestWhileIdle(true);
		 85             connectionPool
		 86                 .setMinEvictableIdleTimeMillis(minEvictableIdleTimeMillis);
		 87             
		 88             // Start the Evictor
		 89             connectionPool
		 90                 .setTimeBetweenEvictionRunsMillis(timeBetweenEvictionRunsMillis);

### 参考文档
* [GenericObjectPool的java doc](http://commons.apache.org/proper/commons-pool/api-1.6/org/apache/commons/pool/impl/GenericObjectPool.html)