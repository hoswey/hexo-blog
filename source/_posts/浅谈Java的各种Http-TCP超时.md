---
title: 浅谈Java的各种Http/TCP超时
date: 2019-07-17 12:26:48
tags:
---

## 背景

在远程调用的世界里，Timeout的情况非常常见，几乎每段时间就会听到几个同事关于Timeout各种情况的讨论，偶尔的会出现不同开发语言间的同事的讨论，例如read timeout, 是否在C和Java是同一种情况？

对于Java，各种远程调用，http,hessian,dubbo什么的，抛个timeout异常也是常见的事情，所以本文从Java到操作系统层面尝试说明常见的各种Timeout

## 主要内容

### 现象

对于Java开发来说，最常见的异常莫过于SocketTimeoutException，从异常日志，一般会有两种情况

- connect timed out
- read timed out

```java
Caused by: java.net.SocketTimeoutException: connect timed out
        at java.net.PlainSocketImpl.socketConnect(Native Method)
        at java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:345)
        at java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)
        at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)
        at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)
        at java.net.Socket.connect(Socket.java:589)

java.net.SocketTimeoutException: Read timed out
        at java.net.SocketInputStream.socketRead0(Native Method)
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
        at java.net.SocketInputStream.read(SocketInputStream.java:170)
        at java.net.SocketInputStream.read(SocketInputStream.java:141)
```

### 原理 connect timed out

"connect timed out"从字面上看就是连接的超时时间，那么超时时间是怎么控制的？

#### java.net.Socket

{% plantuml %}

Socket -> Socket: connect(timeout)
Socket -> SocksSocketImpl: connect
SocksSocketImpl -> AbstractPlainSocketImpl: connect
AbstractPlainSocketImpl  -> AbstractPlainSocketImpl:socketConnect
AbstractPlainSocketImpl -> PlainSocketImpl: socketConnect(native方法)
{% endplantuml %}

从Sokcet的connect方法可以看出，timeout参数会一致往下传递，最后到了PlainSocketImpl.socketConnect的native方法， java native方法是否真的很神秘？也不神秘，让我们一起看下JVM底层的实现,以下是jdk8-openjdk的源码

#### PlainSocketImpl.c

{% plantuml %}
PlainSocketImpl.c  -> PlainSocketImpl.c: Java_java_net_PlainSocketImpl_socketConnect

PlainSocketImpl.c -> linux_close.c: NET_Connect(timeout <= 0)
linux_close.c -> System API: connect

PlainSocketImpl.c -> linux_close.c: NET_Poll(timeout > 0 && ifndef USE_SELECT )
linux_close.c -> System API: poll

PlainSocketImpl.c -> linux_close.c: NET_Select(timeout > 0 && !ifndef USE_SELECT )
linux_close.c -> System API: Select


{% endplantuml %}


以下只截取部分重要的源码， 从源码上看，没设置超时时间时，jvm采用 connect的传统阻塞式方式，反之,则采用select/poll非阻塞式的方式, 由于poll/select都是得采用轮询的方式，在客户端没有设置的时候，采用轮询会带来不必要的开销

```c
JNIEXPORT void JNICALL
Java_java_net_PlainSocketImpl_socketConnect(JNIEnv *env, jobject this,
                                            jobject iaObj, jint port,
                                            jint timeout)
{

if (timeout < 0 ) {
   connect_rv = NET_Connect(fd, (struct sockaddr *)&him, len);
	...
}
else {
 
#ifndef USE_SELECT
                {
                    struct pollfd pfd;
                    pfd.fd = fd;
                    pfd.events = POLLOUT;

                    errno = 0;
                    connect_rv = NET_Poll(&pfd, 1, timeout);
                }
#else
                {
                    fd_set wr, ex;
                    struct timeval t;

                    t.tv_sec = timeout / 1000;
                    t.tv_usec = (timeout % 1000) * 1000;

                    FD_ZERO(&wr);
                    FD_SET(fd, &wr);
                    FD_ZERO(&ex);
                    FD_SET(fd, &ex);

                    errno = 0;
                    connect_rv = NET_Select(fd+1, 0, &wr, &ex, &t);
                }
#endif

            } 

            if (connect_rv == 0) {
                JNU_ThrowByName(env, JNU_JAVANETPKG "SocketTimeoutException",
                            "connect timed out");

                /*
                 * Timeout out but connection may still be established.
                 * At the high level it should be closed immediately but
                 * just in case we make the socket blocking again and
                 * shutdown input & output.
                 */
                SET_BLOCKING(fd);
                JVM_SocketShutdown(fd, 2);
                return;
            }

            /* has connection been established */
            optlen = sizeof(connect_rv);
            if (JVM_GetSockOpt(fd, SOL_SOCKET, SO_ERROR, (void*)&connect_rv,
                               &optlen) <0) {
                connect_rv = errno;
            }
        }
    }

```

### 原理 Read timed out

从下面这个序列图看， read timedout的原理就是通过系统调用 poll, 传入对应的socket文件句柄，在timeout时间内没有数据返回

{% plantuml %}
SocketInputStream -> SocketInputStream: read(timeout)
SocketInputStream -> AbstractPlainSocketImpl: acquireFD
SocketInputStream -> SocketInputStream.c: Native Method socketRead0(fd, timeout)
SocketInputStream.c -> linux_close.c: NET_Timeout(timeout)
linux_close.c -> System API: poll(fd, POLLIN | POLLERR, timeout)

{% endplantuml %}

当调用NET_Timeout没返回任何数据的时候, 根据情况会抛出 SocketTimeoutException或者SokcetException, 这个SocketTimeoutException就是我们经常遇到的read timed out

```c
if (timeout) {
    nread = NET_Timeout(fd, timeout);
    if (nread <= 0) {
        if (nread == 0) {
            JNU_ThrowByName(env, JNU_JAVANETPKG "SocketTimeoutException",
                        "Read timed out");
        } else if (nread == JVM_IO_ERR) {
            if (errno == EBADF) {
                 JNU_ThrowByName(env, JNU_JAVANETPKG "SocketException", "Socket closed");
             } else {
                 NET_ThrowByNameWithLastError(env, JNU_JAVANETPKG "SocketException",
                                              "select/poll failed");
             }
        } else if (nread == JVM_IO_INTR) {
            JNU_ThrowByName(env, JNU_JAVAIOPKG "InterruptedIOException",
                        "Operation interrupted");
        }
        if (bufP != BUF) {
            free(bufP);
        }
        return -1;
    }
}
```    

## 总结

1. "connect timed out" 是在指定时间内TCP连接未创建成功时jdk抛出的异常
2. "Read timed out"是在调用socketread后，指定时间内未收到响应时 jdk抛出的异常。
