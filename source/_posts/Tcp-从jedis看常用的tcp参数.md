---
title: Tcp - 从jedis看常用的tcp参数
date: 2018-05-17 09:19:35
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

#### 核心tcp参数设置

```java

socket.setReuseAddress(true);
socket.setKeepAlive(true); // Will monitor the TCP connection is
// valid
socket.setTcpNoDelay(true); // Socket buffer Whetherclosed, to
// ensure timely delivery of data
socket.setSoLinger(true, 0); // Control calls close () method
socket.setSoTimeout(soTimeout);
```

##### 1 setReuseAddress 

可以看出setReuseAddress本质上就是对SO_REUSEADDR的设置

```java

public void setReuseAddress(boolean on) throws SocketException {
    if (isClosed())
        throw new SocketException("Socket is closed");
    getImpl().setOption(SocketOptions.SO_REUSEADDR, Boolean.valueOf(on));
}

```

**SO_REUSEADDR**主要的作用([其它作用](https://stackoverflow.com/questions/14388706/socket-options-so-reuseaddr-and-so-reuseport-how-do-they-differ-do-they-mean-t?answertab=active#tab-top))就是对TIME_WAIT连接的重用，对于jedis,作者应该是希望客户端重启后，重新创建到redis服务器的tcp连接能够重用之前的接口。因为客户端重启时，活跃的jedis连接由于时客户端主动关闭，会持续2*MSL的TIME_WAIT状态，在此期间会一直占用着客户端端口。
MSL的大小可通过以下命令查看

```bash
sysctl -a | grep tcp_fin_timeout
tcp_fin_timeout = 60
```

##### 2 setKeepAlive

**SO_KEEPALIVE**是tcp利用心跳机制保持连接的存活，假如连接已经断开，则会响应错误码Broken Pipe给上层应用,由于keepalive发起心跳包的开始实际比较迟，对于调整后的linux操作系统仍然需要5分钟，数值过小又会导致过多无用的包发出，5分钟的时间长度发出后，通常也会由于各种各样的原因连接早已被销毁，例如客户端服务器端的连接经过了lvs，haproxy各种各样的代理，代理服务器本身也会有个keepalive的最长时间。
一般业务对于SO_KEEPALIVE的依赖比较小，假如需要保持连接的话，会自己进行一些存活性的维持和判断，例如

1. 定期小周期发送心跳包keep alive
2. 在使用连接前进行判断，例如连接池通常采用的testOnBorrow, testOnReturn之类的机制。

```bash
sudo sysctl -a | grep tcp_keepalive   
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_time = 300 //通常默认值为7200
```

##### 3 setTcpNoDelay
TCP_NODELAY的作用就是禁用Nagle，减少由于小包带来的推迟发送的延迟
> If set, disable the Nagle algorithm.  This means that segments are always sent  as  soon  as  possible,
even  if  there is only a small amount of data.  When not set, data is buffered until there is a suffi‐
cient amount to send out, thereby avoiding the frequent sending of small packets, which results in poor
utilization of the network.  This option is overridden by TCP_CORK; however, setting this option forces
an explicit flush of pending output, even if TCP_CORK is currently set.

##### 4 setSoLinger(true, 0)
这个选项的设置就是jedis关闭的时候直接发送RST,而不是按照正常的4次挥手的关闭流程，避免了客户端TIME_WAIT的情况，但不是一个很好的pratice,因为服务端会只会收到RST,这里留下一个问题，redis是如何处理rst的，是否是直接忽略，不然就会堆积很多错误日志？

##### 5 socket.setSoTimeout(soTimeout)
SO_TIMEOUT的作用就是设置以下三个socket操作的超时时间，很明显，ServerSocket.accept()是针对服务端而言，DatagramSocket.receive()是针对UDP,在这里jedis生效的是SocketInputStream.read()，作用就是jedis发送命令后，开始读redis请求的响应时间。

```java

ServerSocket.accept();
SocketInputStream.read();
DatagramSocket.receive();

```

## 总结
tcp是一个非常复杂的网络协议，但对于“客户端”的tcp编程其实也不是特别难，从jedis在tcp的使用上看，直接用了blocking io,同时设置了5个参数，就满足了大部分场合的使用，对于一般互联网的服务，也只是使用多一个池化JedisPool而已。
