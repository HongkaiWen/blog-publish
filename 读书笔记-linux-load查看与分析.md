---
title: 读书笔记-linux load查看与分析
date: 2016-03-05 17:23:41
tags: 读书笔记
---

读书笔记 - 《大型分布式网站架构设计与实践》

**LOAD**

定义:  特定时间间隔内运行队列中的平均数，       一个线程如果满足下面三个条件，就会处于运行队列中。
 * 没有处于IO等待状态
 * 没有主动进入等待状态（wait）
 * 没有被终止。

举例说明：8核cpu 运行16根线程，如果线程平均分配，每个cpu运行队列中有两个线程在运行，如果此状态持续一分钟，那么这一分钟内系统的load值就是2.
 
判断条件：如果load不大于3，系统负载正常；大于5则负载过高。
 
查看方法：uptime\top\dstat

### uptime

``` bash
[root@xxx ~]# uptime
 17:27:03 up  1:02,  2 users,  load average: 0.00, 0.03, 0.05
```

### top

``` bash
[root@xxx ~]# top
top - 17:27:28 up  1:03,  2 users,  load average: 0.25, 0.09, 0.06
Tasks: 114 total,   2 running, 112 sleeping,   0 stopped,   0 zombie

```

### dstat

``` bash
[root@xxx ~]# dstat -l
---load-avg---
 1m   5m  15m
0.15 0.08 0.06
0.15 0.08 0.06
```
