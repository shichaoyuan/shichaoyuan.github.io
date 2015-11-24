---
layout: post
title:  "如何高仿Instagram的热门发现Tab"
date:   2015-02-12 19:40:14
tags: engineering
---

# 1

最近在做的项目类似Instagram，其中第二个Tab基本上完全一致……

如何做呢？

首先看看INS是如何做的。

狂刷INS的第二个Tab，发现其中的feed分了两类：“全球热门项目”和“根据你关注的用户”；点击进入feed终端页，感觉内容你预先加载完的（除了照片）。

申请一个开发者账号，调用一下https://api.instagram.com/v1/media/popular?access_token=ACCESS-TOKEN

发现其返回的json包含feed的完整信息。

# 2

我们的需求：

1. 下拉刷新，根据权重随机呈现；
2. 上滑翻页，分页读取；
3. 多个内容源，比如：运营筛选、算法推荐、地理位置；
4. 可以很用容易改变不同内容源的配比。

还有不要太慢 嘻嘻

# 3

总体设计：

1. 生产内容源；
2. 配比内容源，更新线上内容库；
3. 读取线上内容库，根据权重随机呈现。

其中使用Redis做为线上内容库。

# 4

生产内容源

目前有两个：一个mahout的协同过滤推荐算法，一个运营人员筛选。

没什么需要解释的。

# 5

配比内容源，更新线上内容库；

一个定时任务，从不同的内容源中选择不同的配备的内容，然后推送到Redis中。

其中如何设定权重是一个比较有趣的问题，先留坑。

# 6

读取线上内容库，根据权重随机呈现

Redis中数据的组织形式是，userId对应一个redisList，redisList中数据项的序列化和反序列化使用protobuf。

目前设想单个user对应feed的数量为一万左右，如果一次全部读取，并发量较高的时候，程序GC的负担会比较重，所以使用了分段读取的方法——[RedisListIterator.java](https://github.com/shichaoyuan/snippets/blob/master/RedisListIterator.java)。

如何根据权重随机选择，参考之前的文章——[关于sample算法](http://shichaoyuan.github.io/algorithm/2014/12/28/sample_algorithm.html)。

至于分页，用Reids的List也很容易解决。

# 7

该接口的性能基本上严重依赖Redis的性能；

随着用户的增加，Reids集群的容量也得成正比增加。
