---
title: 填坑日记之@Scheduled and RestTemplate
date: 2018-04-29 23:35:41
tags: 填坑日记
---

**越诡异的问题，原因越低级   --  我**

**没用过的工具先了解清楚，否则别瞎用，非要用就要有对此负责的能力 — 我**



### 背景



近几个月基于k8s做发布平台，清明刚刚上线，很忙。

难得本周三早点下班，陪陪老婆孩子。

有平台用户在群里叫：发布的程序一直停留在构建中。

21：00返回公司紧急处理。

挖坑的小弟通过重启应用解决问题后回家睡觉了。

我陷入深深的恐惧，因为原因还没找到。

我留在公司，在开发环境模拟用户的操作去重现问题。



### 代码逻辑

代码里有若干定时任务

- 有一个是call其他模块接口的
- 有一个是与jenkins做交互的，一直停留在构建中，说明这块有问题



### 猜测

- jenkins性能问题，导致程序调用jenkins接口hang住
- 数据库问题导致个别程代码序假死
- 主机性能问题，重点查看IO因为之前吃过这种亏

所有的猜测都是怀疑与jenkins做交互的那个定时任务的某一环节卡住了。



### 求证

拉jvm的栈日志。

间隔一段时间，连续拉取了5次栈信息。人困马乏，懒得分析，直接丢给 http://fastthread.io/ 去分析。没有任何警告信息。在栈dump里搜公司名，发现五次栈都是在执行某一段代码，而这段代码，是存在于另外一个定时任务里的，定位在了调用http接口的方法上。

第一反应，没设超时时间哇，果不其然。而且居然在定时任务里写这种代码：

```java
RestTemplate restTemplate=new RestTemplate();
restTemplate.exchange...
```

我当时也是一口老血啊......

回过头来想，这个调用接口的方法只是查询状态的，怎么会影响构建的job呢。

想不通。



### 再猜测

定时任务使用的是如下方式

```java
@EnableScheduling
@Scheduled(fixedDelay = xxx)
```

是不是这玩意有啥坑啊（这东西我没用过，也不是我引进来的）

默认给的什么类型的线程池啊？



### 求证

https://docs.spring.io/spring/docs/4.0.x/spring-framework-reference/htmlsingle/#scheduling

```
an executor may be single-threaded or even synchronous
```

自己也写了两个schedule验证了一下，确实是默认就给了一个线程，一个schedule卡住会影响其他的。



### 解决办法

- 自定义schedule的线程池，很简单
- resttemplate 配置默认的超时时间（这块使用的apache httpclient管理http连接池，设置了很多参数，比较重要的几个：connect timeout， read timeout， 还有从连接池取连接的最大等待时间，这些都是之前碰过的坑一定不能忘）



### 总结

两个低级的错误，引发的一系列诡异问题。

排查错误的过程也没什么技术含量，只是想把产品做好，把用户服务好，坚持不懈的填坑而已。

发现自己团队内有能力做这一系列排查的人还没有，所以把过程记录下来。

团队建设，任重道远。





