---
title: Spring - Rabbitmq bug排查
date: 2018-04-24 09:35:23
tags:
---

## 背景
项目同时连接多个rabbitmq,使用spring-rabbit进行连接，版本号为1.4.6.RELEASE

某天发现rabbitmq服务器日志错误日志出现大量
>Channel error on connection <0.3019.46> (1.1.1.1:41724 -> 1.2.3.4:5672, vhost: '/', user: 'xunhuan\_prod'), channel 31: operation queue.declare caused a channel exception not_found: "no queue 'wolfkill.groups.online.time.queue' in vhost '/'"

### rabbitmq架构

两个rabbitmq集群，使用不同的virtual host

|集群|Virtual Host|Queue|
|---|---|---|
|A|service_A| A1, A2|
|B| /|wolfkill.groups.online.time.queue|

## 故障排查
### 检查spring xml配置文件
  - 两个rabbitmq配置都已经分开
  - queue和exchange都全部加上了[declared-by](https://docs.spring.io/spring-amqp/reference/htmlsingle/#conditional-declaration)

### 检查源代码

#### SimpleMessageListenerContainer
初始化的时候判断当前RabbitAdmin的数量，假如不为空的话，就赋值给成员变量RabbitAdmin,看到这里时已经感觉有点不妥，这也太过随意了吧😮，该项目接入了两个mq,所以RabbitAdmin对应着有两个。
[源代码传送门](https://github.com/spring-projects/spring-amqp/blob/f9586b5de7e852033a5ece2cb229f3cac740e053/spring-rabbit/src/main/java/org/springframework/amqp/rabbit/listener/SimpleMessageListenerContainer.java#L685)

```java
@Override
protected void doStart() throws Exception {
	super.doStart();
	if (this.rabbitAdmin == null && this.getApplicationContext() != null) {
	  //初始化时会随机拿第一个RabbitAdmin设置为当前的变量以供后续使用，但当项目有多个RabbitAdmin时则可能设置错误
		Map<String, RabbitAdmin> admins = this.getApplicationContext().getBeansOfType(RabbitAdmin.class);
		if (!admins.isEmpty()) {
			this.rabbitAdmin = admins.values().iterator().next();
		}
	}
```

在doStart之后执行的代码，拿了不确定的RabbitAdmin继续declare Queue(调用Rabbitmq Broker api).
[源代码传送门](https://github.com/spring-projects/spring-amqp/blob/f9586b5de7e852033a5ece2cb229f3cac740e053/spring-rabbit/src/main/java/org/springframework/amqp/rabbit/listener/SimpleMessageListenerContainer.java#L963)


```java
	private synchronized void redeclareElementsIfNecessary() {
		try {
			ApplicationContext applicationContext = this.getApplicationContext();
			if (applicationContext != null) {
				Set<String> queueNames = this.getQueueNamesAsSet();
				Map<String, Queue> queueBeans = applicationContext.getBeansOfType(Queue.class);
				for (Entry<String, Queue> entry : queueBeans.entrySet()) {
					Queue queue = entry.getValue();
					//rabbitAdmin.getQueueProperties会调用channel.queueDeclarePassive,所以这个时候会拿了错误的RabbitAdmin进行创建，所以该处是问题所在
					if (queueNames.contains(queue.getName()) &&
							this.rabbitAdmin.getQueueProperties(queue.getName()) == null) {
						if (logger.isDebugEnabled()) {
							logger.debug("At least one queue is missing: " + queue.getName()
									+ "; redeclaring context exchanges, queues, bindings.");
						}
						this.rabbitAdmin.initialize();
						return;
					}
				}
			}
		}
```

#### 检查新版本代码
[源代码传送门](https://github.com/spring-projects/spring-amqp/blob/233aa11cf9046fb8c2c3c5a373d09d9e645aa2b3/spring-rabbit/src/main/java/org/springframework/amqp/rabbit/listener/SimpleMessageListenerContainer.java#L801)

检查最新的版本1.7.3.RELEASE, 最新版本已经加了条件判断，当只有一个RabbitAdmin时才会设置。

```java
	@Override
	protected void doStart() throws Exception {
		if (this.rabbitAdmin == null && this.getApplicationContext() != null) {
			Map<String, RabbitAdmin> admins = this.getApplicationContext().getBeansOfType(RabbitAdmin.class);
			if (admins.size() == 1) {
				this.rabbitAdmin = admins.values().iterator().next();
			}
```

**所以该问题的原因是spring rabbitmq在项目同时存在多个RabbitAdmin时，误用了RabbitAdmin, RabbitAdmin就是分别对应着不同的Rabbitmq集群，在该场景下，用了RabbitAdmin A去创建原本应该创建在B上的Virtual Host为“/”的Queue"wolfkill.groups.online.time.queue".**

## 总结
1. 该问题是spring-rabbitmq模块的bug,搞版本已经解决
1. 使用开源工具时，尽可能的使用最新的版本。