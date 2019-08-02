---
title: 浅谈Nginx&Ribbon的容错处理
tags:
---

## 背景

最近由于服务要调用某个依赖的http服务，该依赖服务部署在阿里云主机上，阿里云主机无法安装lvs,导致单点问题，所以针对该依赖服务，特意引入了Ribbon,解决了单点问题。


以下是带LVS和无LVS的情况

![翻墙中](https://drive.google.com/uc?export=view&id=15uwFB0SdPRB1XA8XeEol0WpHXcji1cv5)

**传统方式**nginx前面都挂了个lvs,增加了ip接入点的高可用，当没LVS的情况下, 客户端直直接把请求轮询分别调用两个nginx, 这个时候假如一个nginx挂了，那么一半的请求就会失败。

## 解决方式

那么怎么解决这种问题呢？

1. 服务发现，这也是最近今年各种框架技术采用的，例如dubbo, neflix eureka，如果nginx注册到注册中心，例如zk,客户端通过zk获取nginx节点，这样单台nginx挂了的时候不影响服务。
2. 非服务发现，客户端增加容错策略，记录一些统计信息，例如延迟，失败率，通过延迟和失败率把问题节点排除掉，这样加入某台nginx挂了的时候，客户端几次的失败调用，就能够侦测到某台机器出现了问题。

这里重点分析的是第二种方法，我们简称为客户端容错，客户端容错有比第一种额外的优势，这点稍后讲到。从这个容错策略，很容易的就可以联想到nginx的upstream

```
upstream backend {
    server backend1.example.com weight=5;
    server 127.0.0.1:8080       max_fails=3 fail_timeout=30s;
    server unix:/tmp/backend3;

    server backup1.example.com  backup;
}
```

### Nginx容错

这里主要的参数是

[max_fails](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#max_fails)和
[fail_timeout](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#fail_timeout)，这两个参数控制了nginx作为客户端时，如何容错，假如在fail_timeout的时间里，这台服务器出现了max_fails次失败，那么就会把这台服务从自己的目标列表里移除，fail_timeout秒后再恢复。


### Ribbon容错


