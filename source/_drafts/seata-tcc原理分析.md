title: seata tcc原理分析
date: 2020-11-23 12:42:40
tags:
---
## 背景
由于项目最近准备引入tcc分布式事物解决方案，调研了seata,看起来seata的tcc prepare-comfirm-rollback的模式契合业务场景，但由于该工具在业界使用不是很高，软件质量也非工业级的水平，所以需要了解该框架的核心链路工作流程，保持遇到问题能够快速定位解决的能力，由于项目目前主要用了dubbo的rpc框架，所以主要分析了dubbo+tcc的应用场景，该文章主要解决问题

1. tcc是怎么和spring结合
1. tcc补偿流程的执行原理，seata是如何记录持久化任务，进行补偿的
1. tcc有什么常见的坑

## seata tcc的主要流程
### 两阶段提交的流程

[官方文档](http://seata.io/zh-cn/docs/dev/mode/tcc-mode.html)上已经说明了TCC的流程
![](http://seata.io/img/seata_tcc-1.png)

下面以[seata-example](https://github.com/seata/seata-samples/tree/master/tcc/springboot-tcc-sample)下的例子详细描述其交互流程

{% plantuml %}
{% endplantuml %}

### TM
TM需要负责整个

##






## 问题
1. 数据库存储部分没使用连接池
2. TwoPhaseBusinessAction.name 需要唯一，为什么框架不默认类名+方法名？