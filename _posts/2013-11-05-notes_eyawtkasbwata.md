---
layout: post
title:  "Reading notes: Everything You Always Wanted to Know About Synchronization but Were Afraid to Ask"
date:   2013-11-05 16:30:01
tags: concurrency
---

[Everything You Always Wanted to Know About Synchronization but Were Afraid to Ask](http://sigops.org/sosp/sosp13/papers/p33-david.pdf)，SOSP'13上的一篇文章，如果对concurrent感兴趣，那么推荐读一下。

扩展软件系统到多核很难很重要，关键在于synchronization。

> From the Greek "syn", i.e., *with*, and "khronos", i.e., *time*, synchronization denotes the act of coordinating the timeline of a set of processes. (供扯淡使用) 

也就是说，设计一个synchronization scheme，随着核数的增加，性能不下降。但是由于软件层面的同步与硬件层面的同步是相互独立的，所以很难提出普适性的设计，很难评估新的研究成果。总的来说同步的可扩展性主要在于硬件层面。

> Results of this study can be used to help predict the cost of a synchronization scheme, explain its behavior, design better schemes, as well as possibly improve future hardware design.

本文有几个重要的结论：

1.  crossing sockets significantly impacts synchronization, regardless of the layer, e.g., cache coherence, atomic operations, locks. Message passing can be viewed as a way to reduce sharing as it enforces partitioning of the shared data. However, it comes at the cost of lower performance (than locks) on a few cores or low contention.

2.  non-uniformity affects scalability even within a single-socket many-core, i.e., synchronization on a Sun Niagara 2 scales better than on a Tilera TILE-Gx36. 

3. spin locks should be preferred over more complex locks. Complex locks have a lower uncontested performance, a larger memory footprint, and only outperform spin locks under relatively high contention.

4. implementing multi-socket coherence using broadcast or an incomplete directory (as on the Opteron) is not favorable to synchronization.

另外本文还贡献了一个跨平台的同步工具包[SSYNC](http://lpd.epfl.ch/site/ssync)。
