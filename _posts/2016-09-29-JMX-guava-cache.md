---
layout: post
title: 使用JMX操作Guava Cache
date: 2016-09-29 17:05:20 +08:00
tags: java
---

## 0

由于拜读了[《Guava Cache小记》](http://calvin1978.blogcn.com/articles/guava-cache小记.html)，发现`refreshAfterWrite`特性可以很优雅的解决笔者最近遇到的问题，所以开始在项目中用来起来。

由于刚开始用，所以想要监控一下缓存的使用状况，首先想到的周期性打印`CacheStats`到日志，但是感觉这个日志的意思不大，之后很可能还得把这部分代码删掉，所以就想到用JMX查询，而且还可以定义一些简单的操作。

## refresh

值得一提的是，Guava Cache“保证单线程回源”特性解决了缓存失效风暴问题，是一个亮点，但是笔者此处的需求更多的是将本地缓存做为服务降级的兜底数据，另外降低后端redis集群的压力，所以更多利用了“后台定时刷新数据”特性。

异步刷新需要重写`reload`方法，具体实现可以参考[wiki](https://github.com/google/guava/wiki/CachesExplained#refresh)：

```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumSize(1000)
       .refreshAfterWrite(1, TimeUnit.MINUTES)
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) { // no checked exception
               return getGraphFromDatabase(key);
             }

             public ListenableFuture<Graph> reload(final Key key, Graph prevGraph) {
               if (neverNeedsRefresh(key)) {
                 return Futures.immediateFuture(prevGraph);
               } else {
                 // asynchronous!
                 ListenableFutureTask<Graph> task = ListenableFutureTask.create(new Callable<Graph>() {
                   public Graph call() {
                     return getGraphFromDatabase(key);
                   }
                 });
                 executor.execute(task);
                 return task;
               }
             }
           });
```

## 启用JMX

在tomcat中启用jmx需要添加一些启动参数：

```plain
-Dcom.sun.management.jmxremote.port=${MY_JMX_PORT}
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
-Djava.rmi.server.hostname=127.0.0.1
```

因为笔者需要远程访问，所以修改一下：

```plain
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=${MY_JMX_PORT}
-Dcom.sun.management.jmxremote.authenticate=true
-Dcom.sun.management.jmxremote.password.file=${MY_JMX_CONF}/jmxremote.password
-Dcom.sun.management.jmxremote.access.file=${MY_JMX_CONF}/jmxremote.access
-Dcom.sun.management.jmxremote.ssl=false
-Djava.rmi.server.hostname=${PUBLIC_IP}
```

jmxremote.access文件样例：

```plain
monitorRoleUser   readonly
controlRoleUser   readwrite \
              create javax.management.monitor.*,javax.management.timer.* \
              unregister
```

jmxremote.password文件样例：

```plain
monitorRoleUser  pass1
controlRoleUser  pass2
```

配置完后，重启Tomcat，就可以通过jconsole连接上，前几个tab页是基本的监控信息，最后一个tab页MBean提供了查看、操作的GUI交互接口。

## GuavaCache MBean

看到一个gist写的不错，直接拿来用了：

{% gist shichaoyuan/1581efc73926b87fa086a28d42589de4 %}

首先定义接口`GuavaCacheMXBean`，实现类`GuavaCacheMXBeanImpl`只是对`LoadingCache`做一个包装，最后实现了一个注册的静态工具类。

下面看一下注册的流程：

```java
//注册MBean的名字，需要满足格式
String name = String.format("%s:type=%s,name=%s",
        cache.getClass().getPackage().getName(), cache.getClass().getSimpleName(), cacheName);
ObjectName mxBeanName = new ObjectName(name);

Object mBean = new GuavaCacheMXBeanImpl<K, V>(cache);

//获取MBeanServer
MBeanServer server = ManagementFactory.getPlatformMBeanServer();

if (!server.isRegistered(mxBeanName)) {
    //注册
    server.registerMBean(mBean, mxBeanName);
}
```