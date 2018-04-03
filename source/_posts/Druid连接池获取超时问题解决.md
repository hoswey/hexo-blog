---
title: Druid连接池获取超时问题解决
date: 2018-04-03 08:45:02
tags:
---

## 背景
线上某台机器出现频繁的获取mysql连接超时

## 排查步骤
1. 通过日志可以看出服务时获取连接池超时
```
2017-07-14 13:10:01,973 [DubboServerHandler-10.25.65.225:20900-thread-174] ERROR com.alibaba.dubbo.rpc.filter.ExceptionFilter (ExceptionFilter.java:87) -  [DUBBO] Got unchecked and undeclared exception which called by 10.25.68.15. service: com.bilin.user.dynamic.service.IUserDynamicService, method: queryNewestDynamicMsgByUser, exception: org.springframework.jdbc.CannotGetJdbcConnectionException: Could not get JDBC Connection; nested exception is com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 10000, active 0, dubbo version: 2.4.9, current host: 10.25.65.225
org.springframework.jdbc.CannotGetJdbcConnectionException: Could not get JDBC Connection; nested exception is com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 10000, active 0
        at org.springframework.jdbc.datasource.DataSourceUtils.getConnection(DataSourceUtils.java:82)
        at org.springframework.orm.ibatis.SqlMapClientTemplate.execute(SqlMapClientTemplate.java:183)
        at org.springframework.orm.ibatis.SqlMapClientTemplate.executeWithListResult(SqlMapClientTemplate.java:220)
        at org.springframework.orm.ibatis.SqlMapClientTemplate.queryForList(SqlMapClientTemplate.java:267)
        at com.bilin.sdk.dao.base.BaseDAO.queryForList(BaseDAO.java:27)
Caused by: com.alibaba.druid.pool.GetConnectionTimeoutException: wait millis 10000, active 0
        at com.alibaba.druid.pool.DruidDataSource.getConnectionInternal(DruidDataSource.java:1071)
        at com.alibaba.druid.pool.DruidDataSource.getConnectionDirect(DruidDataSource.java:898)
        at com.alibaba.druid.filter.FilterChainImpl.dataSource_connect(FilterChainImpl.java:4544)
        at com.alibaba.druid.filter.stat.StatFilter.dataSource_getConnection(StatFilter.java:661)
```
2. 猜测可能的原因
  - 并发量太高
    从数据上看，每秒不到1k请求，后端mysql服务器的负载也不高，所以这个原因基本排除
  - 连接池连接数设置过低
    业务设置的minIdel 100, maxConnection 300, 所以这个原因基本排除。
  - mysql服务故障
    部署在别的机器上的服务，还有使用同一个db实例的其它业务没出现异常情况，所以这原因概率不大
  - mysql链接设置过低
3. 由于没有比较明显的可能性，所以只能看下源码(版本1.0.5)找下具体的原因。
  [DruidDataSource](https://github.com/alibaba/druid/blob/1.0.5/src/main/java/com/alibaba/druid/pool/DruidDataSource.java)
  异常的抛出是在1071行
  [throw new GetConnectionTimeoutException(errorMessage);](https://github.com/alibaba/druid/blob/1.0.5/src/main/java/com/alibaba/druid/pool/DruidDataSource.java#L1071)
  由于设置了链接获取的超时时间
```xml
<property name="maxWait" value="10000" />
```
  同时由报错的日志可知，连接等待了10秒获取失败，所以可得知执行的方法时pollLast
```java
            if (maxWait > 0) {
                holder = pollLast(nanos); //L 1021
            } else {
                holder = takeLast();
            }
```
  druid的连接获取，是通过notEmpty和empty两个变量协调线程的同步，pollLast发现没可用连接时，就会notEmpty.await(),同时empty.signal().  emtpy.signal主要唤醒了CreateConnectionThread. 通过了解CreateConnectionThread的源码，发现在某些情况下线程会退出。
**情况1**
```java
  try {
      connection = createPhysicalConnection();
  } catch (SQLException e) {
      LOG.error("create connection error", e);

      errorCount++;
      //当错误次数达到设置值时，breakAfterAcquireFailure设置为true时线程会退出
      if (errorCount > connectionErrorRetryAttempts && timeBetweenConnectErrorMillis > 0) {
          if (breakAfterAcquireFailure) {
              break;
          }

          try {
              Thread.sleep(timeBetweenConnectErrorMillis);
          } catch (InterruptedException interruptEx) {
              break;
          }
      }
  } catch (RuntimeException e) {
      LOG.error("create connection error", e);
      continue;
  } catch (Error e) {
      LOG.error("create connection error", e);
      break;
  }
```
**情况2**
```java
DruidConnectionHolder holder = null;
try {
    holder = new DruidConnectionHolder(DruidDataSource.this, connection);
} catch (SQLException ex) {
    //这里也会退出
    LOG.error("create connection holder error", ex);
    break;
}
```
情况1由于在该项目中没设置breakAfterAcquireFailure，采用默认值false,所以情况1不大可能出现，Error属于不可恢复错误，所以退出也合理。
情况2就有点不合理，单个的SQLException也会导致整个的生存线程结束。
4. 所以接下来主要是确认CreateConnectionThread是否存在异常
通过jstack grep线程名Druid-ConnectionPool-Create（线程有设置线程名是个好习惯）,从grep的结果，发现存在一个CreateConnectionThread，由于业务设置了两个连接池，所以一个是不正常
```bash
# sudo -u www-data jstack 10750  | grep "Druid-ConnectionPool-Create"
"Druid-ConnectionPool-Create-376416626" daemon prio=10 tid=0x00007faab4039800 nid=0x2ae3 waiting on condition [0x00007faac3ec6000]
```
继续grep一下Destroy线程，确认是两个线程，所以Destroy线程正常，进一步确认了CreateConnectionThread线程存在问题。
```bash
# sudo -u www-data jstack 10750  | grep "Druid-ConnectionPool-Des"
"Druid-ConnectionPool-Destory-376416626" daemon prio=10 tid=0x00007faab403a800 nid=0x2ae4 waiting on condition [0x00007faac3e85000]
"Druid-ConnectionPool-Destory-190586441" daemon prio=10 tid=0x00007faad8fb1800 nid=0x2a3d waiting on condition [0x00007faac9dda000]
```
5. 基本确认了是由于CreateConnectionThread不正常结束，所以最后一步就是找寻证据证明线程不正常的结束。通过以上源码，可看到Druid在每一步的异常处理都会记录日志，所以通过日志关键字进行grep,发现在1682行写了个错误日志，**对应到的正式线程退出情况2**.所以问题的原因就是druid在获取mysql连接后创建DruidConnectionHolder时由于网络原因报了MySQLException导致了CreateConnectionThread退出。
```bash
2017-07-14 00:26:59,609 [Druid-ConnectionPool-Create-190586441] ERROR com.alibaba.druid.pool.DruidDataSource$CreateConnectionThread (DruidDataSource.java:1682) - create connection holder error
com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
The last packet successfully received from the server was 0 milliseconds ago.  The last packet sent successfully to the server was 0 milliseconds ago.
        at sun.reflect.GeneratedConstructorAccessor33.newInstance(Unknown Source)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:525)
        at com.mysql.jdbc.Util.handleNewInstance(Util.java:411)
        at com.mysql.jdbc.SQLError.createCommunicationsException(SQLError.java:1121)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3673)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3562)
        at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:4113)
        at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2570)
        at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2731)
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2812)
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2761)
        at com.mysql.jdbc.StatementImpl.executeQuery(StatementImpl.java:1612)
        at com.mysql.jdbc.ConnectionImpl.getTransactionIsolation(ConnectionImpl.java:3352)
        at com.alibaba.druid.filter.FilterChainImpl.connection_getTransactionIsolation(FilterChainImpl.java:347)
        at com.alibaba.druid.filter.FilterAdapter.connection_getTransactionIsolation(FilterAdapter.java:872)
        at com.alibaba.druid.filter.FilterChainImpl.connection_getTransactionIsolation(FilterChainImpl.java:344)
        at com.alibaba.druid.filter.FilterAdapter.connection_getTransactionIsolation(FilterAdapter.java:872)
        at com.alibaba.druid.filter.FilterChainImpl.connection_getTransactionIsolation(FilterChainImpl.java:344)
        at com.alibaba.druid.proxy.jdbc.ConnectionProxyImpl.getTransactionIsolation(ConnectionProxyImpl.java:260)
        at com.alibaba.druid.pool.DruidConnectionHolder.<init>(DruidConnectionHolder.java:92)
        at com.alibaba.druid.pool.DruidDataSource$CreateConnectionThread.run(DruidDataSource.java:1680)
Caused by: java.net.SocketException: Connection reset
        at java.net.SocketInputStream.read(SocketInputStream.java:189)
        at java.net.SocketInputStream.read(SocketInputStream.java:121)
        at com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:114)
        at com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:161)
        at com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:189)
        at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:3116)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3573)
        ... 16 more
```
6. 查看了最新版本1.1.2这部分的代码，发现这部分代码已经重构了，不存在该问题，所以升级版本即可。

## 总结
1. 故障的版本时1.0.5，发布于2014年，属于比较旧的版本，开源工具不可避免的会存在bug,所以需要即时的进行升级
2. 使用开源工具时，尽可能的使用最新的版本。
