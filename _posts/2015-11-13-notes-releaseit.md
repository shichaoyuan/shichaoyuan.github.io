---
layout: post
title: Reading Notes - Release It!
date: 2015-11-13 09:05:20
tags: engineering
---

[Release It!](http://book.douban.com/subject/2065284/)

非常值得阅读的一本书，有些内容是大家都很熟悉的，有些是很少考虑的。

在此把书中提到的Patterns汇总罗列了一下~


## Stability

### Use Timeouts

1. HTTP/TCP设置Timeout；
  * Connection Timeout
  * Socket Timeout
  * TIME_WAIT。
2. 数据库操作设置timeout；
3. 线程wait或tryLock设置timeout。

处理超时问题，可以尝试queue-and-retry。关于retry的实现可以借鉴[Recurrent](https://github.com/jhalterman/recurrent)

> Apply to **Integration Points**, **Blocked Theads**, and **Slow Responses**
> 
> Apply to recover from unexpected failures
> 
> Consider delayed retries

### Circuit Breaker

“保险丝”模式

这个模式有个非常著名的实现——[Hystrix](https://github.com/Netflix/Hystrix)

更重要的是对“保险丝”状态的监控

> Don't do it if it hurts
> 
> Use together with **Timeouts**
> 
> Expose, track, and report state changes

### Bulkheads

“消防卷帘门”模式（义译）

实际上就是对资源的隔离，也就是对故障的隔离。

隔离的粒度是个问题，目前很火的是基于[Docker](https://www.docker.com/)的容器级别隔离。

> Save part of the ship
> 
> Decide whether to accept less efficient use of resources
> 
> Pick a useful granularity
> 
> Very important with shared services model

### Steady State

减少人为干预的同时防范无限增长的数据

使用内存缓存，注意限制大小，可以参考[Guava](https://github.com/google/guava)的CacheBuilder。

不要把日志留在线上机器，可以使用[heka](https://github.com/mozilla-services/heka)、[Kafka](http://kafka.apache.org/)等

> Avoid fiddling
> 
> Purge data with application logic
> 
> Limit caching
> 
> Roll the logs

### Fail Fast

顾名思义，检查输入，不行就挂。

> Avoid **Slow Responses** and **Fail Fast**
> 
> Reserve resources, verify **Integration Points** early
> 
> Use for input validation

### Handshake

列几个例子：

1. [Apache Commons Pool](http://commons.apache.org/proper/commons-pool/)中的testOnBorrow、testWhileIdle。
2. [karyon](https://github.com/Netflix/karyon)中的HealthCheckHandler。
3. [Ribbon](https://github.com/Netflix/ribbon)中的LoadBalancer。

> Create cooperative demand control
> 
> Consider health checks
> 
> Build Handshaking into your own low-level protocols

### Test Harness

测试

> Emulate out-of-spec failures
> 
> Stress the caller
> 
> Leverage shared harnesses for common failures
> 
> Supplement, don’t replace, other testing methods

### Decoupling Middleware

多多了解，慎重选择

1. In-Process Method Calls
  * C Functions
  * Java Calls
  * Dynamic Libs
2. Interprocess Communication
  * Shared Memory
  * Pipes
  * Semaphores
  * Windows Events
3. Remote Procedure Calls
  * DCE RPC
  * DCOM
  * RMI
  * XML-RPC
  * HTTP
4. Message Oriented Middleware
  * MQ
  * Pub-Sub
  * SMTP
  * SMS
5. Tuple Spaces
  * JavaSpaces
  * TSpaces
  * GigaSpaces

> Decide at the last responsible moment
> 
> Avoid many failure modes through total decoupling
> 
> Learn many architectures, and choose among them

##Capacity

**systems thinking**

> Understanding the capacity of any system requires *systems thinking*, as described by Peter Senge in The *Fifth Discipline* —the ability to think in terms of dynamic variables, change over time, and interrelated connections. No one simple formula will produce an all-encompassing “capacity number.” 

监控系统很重要，需要可以对比各种参数的相关性。

> C.A.R. Hoare famously said, “Premature optimization is the root of all evil.” This has often been misused as an excuse for sloppy design. Hoare’s full quote said, “We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil.” His true warning was against chasing small gains at the expense of complexity and development time.

这句话不是Donald Knuth说的么…………

### Pool Connections

各种“池”，必须的。

> Pool connections
>
> Protect request-handling threads
> 
> Size the pools for maximum throughput

### Use Caching Carefully

> Limit cache sizes
> 
> Build a flush mechanisum
> 
> Don't cache trivial objects
> 
> Compare access and change frequency

### Precompute Content

> Precompute content that changes infrequently

### Tune the Garbage Collector

这个题目够写一本“砖头”书了。

详见[Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning Guide](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)> Tune the garbage collector in production
> 
> Keep it up
> 
> Don't pool ordinary objects
