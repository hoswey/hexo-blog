---
title: Http长连接的各种细节
tags:
---

## 背景

项目网关和进程之间采用的是http连接，进程转发有时会出现跨机房的情况，所以通过http keepalive保持长连接是减少延迟的关键，但现实并非完全的长连接。

## 现象

网关和进程间的请求已经明确了http头非close, 由于http版本是1.1， 所以只要值不是"close",那么明确的就是需要keepalive.

```
connection: 
```

但现象是，从连接状态上看，还是出现了TIME_WAIT的情况，10.77.1.24是源地址，10.77.1.60:12003是目标服务，TIME\_WAIT,就是服务端主动关闭了连接

```bash
sudo netstat -apn |  grep "10.77.1.60:12003"
tcp        0      0 10.77.1.24:35720        10.77.1.60:12003        TIME_WAIT   -               
tcp        0      0 10.77.1.24:37130        10.77.1.60:12003        ESTABLISHED 20750/java      
tcp        0      0 10.77.1.24:39472        10.77.1.60:12003        ESTABLISHED 20750/java      
tcp        0      0 10.77.1.24:40880        10.77.1.60:12003        ESTABLISHED 20750/java      
tcp        0      0 10.77.1.24:40962        10.77.1.60:12003        ESTABLISHED 20750/java                           
```

## 排查

### 通过tcpdump排查问题

通过在服务器上执行tcpdump, 抓包分析

```bash
sudo tcpdump -i any host 10.77.1.60 and port 12003
```

{% asset_img ws1.jpg example.jpg %}

发现一个现象

1. 每个连接处理一段时间总会关闭，都是客户端也就是网关主动发起的关闭，从上图看，带了Fin标志的包从客户端发往服务端。
2. 连接持续的时间不等，有60s, 90s, 122s, 151s,都不一定是整数

由于是客户端主动发起的关闭连接，那么排查方向主要从客户端入手，看下有无一些设置时长的判断的逻辑之类，浏览了相关源码，发现并无相关代码。

所以考虑到，会否是服务端发起关闭的请求，客户端被动接受而已。所以看了FIN包的上一个http响应，发现tomcat服务器响应了**"Connection: close"**的header.

{% asset_img ws2.jpg example.jpg %}

所以连接的重用是控制在tomcat端，通常控制http keepalive的方式有

1. 连接idle time.
2. 使用该连接请求的次数。

很明显可以排除第一点，怀疑是第二点的原因。统计了几个有相同情况的情况，发现每个请求都是重用了100次，tomcat就响应了**"Connection: close"**.

### 分析tomcat

[Keepalive配置项](https://tomcat.apache.org/tomcat-8.5-doc/config/http.html#HTTP/2_Support)


**maxKeepAliveRequests**

>The maximum number of HTTP requests which can be pipelined until the connection is closed by the server. Setting this attribute to 1 will disable HTTP/1.0 keep-alive, as well as HTTP/1.1 keep-alive and pipelining. Setting this to -1 will allow an unlimited amount of pipelined or keep-alive HTTP requests. If not specified, this attribute is set to 100.



checkout tomcat源码

```bash
git clone  https://github.com/apache/tomcat.git 
//checkout项目使用到的版本8.5.31
git checkout 18fac60288
```

浏览处理http响应的代码，截取部分源码

Http11Processor

```java
public SocketState service(SocketWrapperBase<?> socketWrapper) {
	if (maxKeepAliveRequests == 1) {
	    keepAlive = false;
	} else if (maxKeepAliveRequests > 0 &&
	        socketWrapper.decrementKeepAlive() <= 0) {
	    keepAlive = false;
	}
}

protected final void prepareResponse() throws IOException {
	
	if (!keepAlive) {
	    // Avoid adding the close header twice
	    if (!connectionClosePresent) {
	        headers.addValue(Constants.CONNECTION).setString(
	                Constants.CLOSE);
	    }
	} else if (!http11 && !getErrorState().isError()) {
	    headers.addValue(Constants.CONNECTION).setString(Constants.KEEPALIVE);
	}
}
```

## 总结

1. tomcat作为服务端，通过maxKeepAliveRequests参数控制keepalive连接数的最大支持的数量，默认值为100.
2. 除了maxKeepAliveRequests这个参数，tomcat还提供keepAliveTimeout，控制连接空闲的时间，默认是20秒。
3. 这两个控制正对应着http1.1规范对于keepalive的定义。



