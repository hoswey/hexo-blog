---
title: Tcp - 从jedis看常用的tcp参数
tags:
---

## 背景
由于微服务的兴起，以及公司内部各种服务间的各种调用，最近也遇过几次线上大量TIME_WAIT的问题，虽然知道怎么解决，但也点燃了我对tcp协议重新温习学习的兴趣。

从何学习呢？重新看下几百多页的《TCP/IP详解》，那太花时间了，以前页看过一次。不如从一些优秀的开源项目里，看下他们是怎么使用tcp建立连接的？那就从jedis开始吧。😃


## 源码分析步骤
### 下载源代码
```bash
git clone https://github.com/xetorthio/jedis.git
git checkout tags/jedis-2.9.0
```

### 分析
从最简单的jedis.set追到底层的tcp网络编程吧

#### 时序图

{% plantuml %}
participant Jedis

Jedis -> Client: set
Client -> Connection: sendCommand
Connection -> Connection: connect

create Socket
Connection -> Socket: setReuseAddress(true)
Connection -> Socket: setKeepAlive(true)
Connection -> Socket: setTcpNoDelay(true)
Connection -> Socket: setSoLinger(true,0)
Connection -> Socket: connect(new InetSocketAddress(host, port), connectionTimeout)
Connection -> Socket: setSoTimeout(soTimeout)


{% endplantuml %}