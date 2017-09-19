---
layout: post
title: 实现高并发的CKMS百分位数算法
date: 2017-09-18 22:00:00 +08:00
tags: java
---

## 0

《[基于Striped64实现DoubleMinUpdater](http://scyuan.info/2017/07/25/Striped64.html)》之前的这篇文章介绍了如何高并发的统计最大最小值，对于百分位数设想使用分桶压缩的办法，但是效果不佳，原因是数值范围不可控。

对于这类流式计算百分位数的场景，我们的需求有两条：1.尽可能少占用内存；2.支持高并发调用。

调研了一下[Quantiles on Streams](https://www.cs.ucsb.edu/~suri/psdir/ency.pdf)，大致分两个流派：1.随机算法，本质上变成了随机采样；2.确定性算法，quantiles can be estimated with precision εn。

最终选择了确定性算法——CKMS，虽然有点儿复杂，但是空间复杂度和偏差都可控。

## CKMS开源实现

[micrometer/micrometer-core/src/main/java/io/micrometer/core/instrument/stats/quantile/CKMSQuantiles.java](https://github.com/micrometer-metrics/micrometer/blob/master/micrometer-core/src/main/java/io/micrometer/core/instrument/stats/quantile/CKMSQuantiles.java)

github上找到的star最多的CKMS算法实现。虽然对`sample`加了`synchronized`关键字，但是这个实现是线程不安全的。坦白说这就是个半成品，不能直接拿来用。

## 改造开源实现

### ReentrantLock

所有接口操作加一把`ReentrantLock`。

[quantiles/quantiles-core/src/main/java/scyuan/quantiles/ckms/CKMSQuantilesMT.java](https://github.com/shichaoyuan/quantiles/blob/master/quantiles-core/src/main/java/scyuan/quantiles/ckms/CKMSQuantilesMT.java)

所有进程都要抢一把锁，看来得进一步优化。

### PriorityBlockingQueue

将`buffer`改为`PriorityBlockingQueue`，入队列一把锁，批量插入压缩使用另外一把锁。

```java
@Override
public void observe(double value) {
    bufferQueue.add(value);
    if (bufferQueue.size() >= bufferMaxSize) {
        if (lock.tryLock()) {
            try {
                insertBatch();
                compress();
            } finally {
                lock.unlock();
            }
        }
    }
}
```

如何测试一下性能呢？

## JMH性能测试

[quantiles/quantiles-benchmarks/](https://github.com/shichaoyuan/quantiles/tree/master/quantiles-benchmarks)

```plain
$ java -jar quantiles-benchmarks.jar  -t 10 scyuan.quantiles.QuantilesTestBenchmark.mt

Benchmark                               Mode       Cnt       Score      Error  Units
QuantilesTestBenchmark.mt              thrpt         5  146692.964 ± 4857.368  ops/s
QuantilesTestBenchmark.mt             sample  40599796      ≈ 10⁻⁴              s/op
QuantilesTestBenchmark.mt:mt·p0.00    sample                ≈ 10⁻⁷              s/op
QuantilesTestBenchmark.mt:mt·p0.50    sample                ≈ 10⁻⁷              s/op
QuantilesTestBenchmark.mt:mt·p0.90    sample                ≈ 10⁻⁷              s/op
QuantilesTestBenchmark.mt:mt·p0.95    sample                ≈ 10⁻⁷              s/op
QuantilesTestBenchmark.mt:mt·p0.99    sample                ≈ 10⁻⁷              s/op
QuantilesTestBenchmark.mt:mt·p0.999   sample                 0.002              s/op
QuantilesTestBenchmark.mt:mt·p0.9999  sample                 0.236              s/op
QuantilesTestBenchmark.mt:mt·p1.00    sample                 3.498              s/op
```

```plain
$ java -jar quantiles-benchmarks.jar  -t 10 scyuan.quantiles.QuantilesTestBenchmark.queue

Benchmark                                     Mode       Cnt        Score        Error  Units
QuantilesTestBenchmark.queue                 thrpt         5  1720758.671 ± 268550.543  ops/s
QuantilesTestBenchmark.queue                sample  74478550       ≈ 10⁻⁵                s/op
QuantilesTestBenchmark.queue:queue·p0.00    sample                 ≈ 10⁻⁷                s/op
QuantilesTestBenchmark.queue:queue·p0.50    sample                 ≈ 10⁻⁷                s/op
QuantilesTestBenchmark.queue:queue·p0.90    sample                 ≈ 10⁻⁶                s/op
QuantilesTestBenchmark.queue:queue·p0.95    sample                 ≈ 10⁻⁴                s/op
QuantilesTestBenchmark.queue:queue·p0.99    sample                 ≈ 10⁻⁴                s/op
QuantilesTestBenchmark.queue:queue·p0.999   sample                 ≈ 10⁻⁴                s/op
QuantilesTestBenchmark.queue:queue·p0.9999  sample                  0.011                s/op
QuantilesTestBenchmark.queue:queue·p1.00    sample                  0.081                s/op
```

根据结果来看，吞吐量和耗时上的改善都是很明显的。性能上满足需求了，但是内存占用堪忧。

```plain
[Stat] # of samples: 53390
[Stat] # of data: 20975400

[Stat] # of samples: 135804
[Stat] # of data: 182952246
```

开源实现中采样对象存储为`Item`，135804个采样点就是4.14MB，另外`LinkedList`又得占用3.1MB“指针”空间。

最大的风险是`PriorityBlockingQueue`的最大长度没有限制，可能占用大量的临时空间。

## 重新实现CKMS

减少内存占用，首先想到的是将`Item`拆分成三个primitive类型的数组，在此选用了[joda-primitives](https://github.com/JodaOrg/joda-primitives)的封装，相对之前`LinkedList<Item>`可以省下三分之二的空间。

修改`LinkedList<Item>`的过程中，发现了这样一行注释，原来该实现对原始论文进行了修改，但是作者并没有写清楚原因，只是 essentially a HACK, and blows up memory, but does "work"，本着对原始论文的尊重，所以笔者决定按照论文重新实现。

```
    private double allowableError(int rank) {
        // NOTE: according to CKMS, this should be count, not size, but this leads
        // to error larger than the error bounds. Leaving it like this is
        // essentially a HACK, and blows up memory, but does "work".
        //int size = count;
        int size = sample.size();
```

[quantiles/quantiles-core/src/main/java/scyuan/quantiles/ckms/CKMSQuantilesPrimitive.java](https://github.com/shichaoyuan/quantiles/blob/master/quantiles-core/src/main/java/scyuan/quantiles/ckms/CKMSQuantilesPrimitive.java)

再测一下：

```plain
$ java -jar quantiles-benchmarks.jar  -t 10 scyuan.quantiles.QuantilesTestBenchmark.primitive

Benchmark                                             Mode       Cnt        Score        Error  Units
QuantilesTestBenchmark.primitive                     thrpt         5  2788688.952 ± 162189.816  ops/s
QuantilesTestBenchmark.primitive                    sample  87161676       ≈ 10⁻⁵                s/op
QuantilesTestBenchmark.primitive:primitive·p0.00    sample                 ≈ 10⁻⁷                s/op
QuantilesTestBenchmark.primitive:primitive·p0.50    sample                 ≈ 10⁻⁷                s/op
QuantilesTestBenchmark.primitive:primitive·p0.90    sample                 ≈ 10⁻⁷                s/op
QuantilesTestBenchmark.primitive:primitive·p0.95    sample                 ≈ 10⁻⁶                s/op
QuantilesTestBenchmark.primitive:primitive·p0.99    sample                 ≈ 10⁻⁴                s/op
QuantilesTestBenchmark.primitive:primitive·p0.999   sample                 ≈ 10⁻⁴                s/op
QuantilesTestBenchmark.primitive:primitive·p0.9999  sample                 ≈ 10⁻³                s/op
QuantilesTestBenchmark.primitive:primitive·p1.00    sample                  0.011                s/op
```

```plain
[Stat] # of samples: 170
[Stat] # of data: 339458916
```

性能提升了，内存占用也控制住了。

接着验证一下正确性：

[quantiles/quantiles-core/src/test/java/scyuan/quantiles/CKMSQuantilesTest.java](https://github.com/shichaoyuan/quantiles/blob/master/quantiles-core/src/test/java/scyuan/quantiles/CKMSQuantilesTest.java)

```plain
CKMSQuantilesQueue
Q(0.5000000, 0.0100000) is 4999655.0000000 (actual 4999999.0000000, off by 0.0000344)
Q(0.9000000, 0.0100000) is 9000001.0000000 (actual 8999999.0000000, off by 0.0000002)
Q(0.9500000, 0.0010000) is 9499955.0000000 (actual 9499999.0000000, off by 0.0000044)
Q(0.9900000, 0.0010000) is 9900061.0000000 (actual 9899999.0000000, off by 0.0000062)
Q(0.9990000, 0.0001000) is 9990043.0000000 (actual 9989999.0000000, off by 0.0000044)
Q(0.9999000, 0.0000100) is 9999075.0000000 (actual 9998999.0000000, off by 0.0000076)
# of samples: 37098
# of data: 10000000

CKMSQuantilesPrimitive
Q(0.5000000, 0.0100000) is 4984782.0000000 (actual 4999999.0000000, off by 0.0015217)
Q(0.9000000, 0.0100000) is 9001126.0000000 (actual 8999999.0000000, off by 0.0001127)
Q(0.9500000, 0.0010000) is 9505663.0000000 (actual 9499999.0000000, off by 0.0005664)
Q(0.9900000, 0.0010000) is 9901740.0000000 (actual 9899999.0000000, off by 0.0001741)
Q(0.9990000, 0.0001000) is 9990283.0000000 (actual 9989999.0000000, off by 0.0000284)
Q(0.9999000, 0.0000100) is 9998967.0000000 (actual 9998999.0000000, off by 0.0000032)
# of samples: 170
# of data: 10000000

CKMSQuantilesMT
Q(0.5000000, 0.0100000) is 4999655.0000000 (actual 4999999.0000000, off by 0.0000344)
Q(0.9000000, 0.0100000) is 9000001.0000000 (actual 8999999.0000000, off by 0.0000002)
Q(0.9500000, 0.0010000) is 9499955.0000000 (actual 9499999.0000000, off by 0.0000044)
Q(0.9900000, 0.0010000) is 9900061.0000000 (actual 9899999.0000000, off by 0.0000062)
Q(0.9990000, 0.0001000) is 9990043.0000000 (actual 9989999.0000000, off by 0.0000044)
Q(0.9999000, 0.0000100) is 9999075.0000000 (actual 9998999.0000000, off by 0.0000076)
# of samples: 37098
# of data: 10000000


CKMSQuantilesOrigin
Q(0.5000000, 0.0100000) is 4999195.0000000 (actual 4999999.0000000, off by 0.0000804)
Q(0.9000000, 0.0100000) is 8999872.0000000 (actual 8999999.0000000, off by 0.0000127)
Q(0.9500000, 0.0010000) is 9499887.0000000 (actual 9499999.0000000, off by 0.0000112)
Q(0.9900000, 0.0010000) is 9900061.0000000 (actual 9899999.0000000, off by 0.0000062)
Q(0.9990000, 0.0001000) is 9990052.0000000 (actual 9989999.0000000, off by 0.0000053)
Q(0.9999000, 0.0000100) is 9999075.0000000 (actual 9998999.0000000, off by 0.0000076)
# of samples: 39348
# of data: 10000000
```

从结果来看是符合错误偏差限制的。如果需要更精确，我们可以调小ε，而不是像原作者那样HACK。

## ThreadLocal

考虑到线程比较多的情况，进一步将一个buffer修改为`ThreadLocal`的buffer

[quantiles/quantiles-core/src/main/java/scyuan/quantiles/ckms/CKMSQuantilesThreadLocal.java](https://github.com/shichaoyuan/quantiles/blob/master/quantiles-core/src/main/java/scyuan/quantiles/ckms/CKMSQuantilesThreadLocal.java)

再测一下：

```plain
$ java -jar quantiles-benchmarks.jar  -t 10 scyuan.quantiles.QuantilesTestBenchmark.threadlocal

Benchmark                                                 Mode       Cnt        Score    Error  Units
QuantilesTestBenchmark.threadlocal                       thrpt            2354176.529           ops/s
QuantilesTestBenchmark.threadlocal                      sample  13258745       ≈ 10⁻⁵            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.00    sample                 ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.50    sample                 ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.90    sample                 ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.95    sample                 ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.99    sample                 ≈ 10⁻⁴            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.999   sample                  0.001            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.9999  sample                  0.034            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p1.00    sample                  0.181            s/op

```

线程数为10的时候，性能反而还降低了一点儿，将线程数增加到100试一下：

```plain
$ java -jar quantiles-benchmarks.jar  -t 100 -f 1 scyuan.quantiles.QuantilesTestBenchmark.threadlocal

Benchmark                                                 Mode        Cnt        Score    Error  Units
QuantilesTestBenchmark.threadlocal                       thrpt             2125265.389           ops/s
QuantilesTestBenchmark.threadlocal                      sample  102052568       ≈ 10⁻⁴            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.00    sample                  ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.50    sample                  ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.90    sample                  ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.95    sample                  ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.99    sample                   0.002            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.999   sample                   0.002            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p0.9999  sample                   0.003            s/op
QuantilesTestBenchmark.threadlocal:threadlocal·p1.00    sample                   0.028            s/op

```

```plain
$ java -jar quantiles-benchmarks.jar  -t 100 -f 1 scyuan.quantiles.QuantilesTestBenchmark.primitive

Benchmark                                             Mode       Cnt        Score    Error  Units
QuantilesTestBenchmark.primitive                     thrpt            2658186.212           ops/s
QuantilesTestBenchmark.primitive                    sample  99924534       ≈ 10⁻⁴            s/op
QuantilesTestBenchmark.primitive:primitive·p0.00    sample                 ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.primitive:primitive·p0.50    sample                 ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.primitive:primitive·p0.90    sample                 ≈ 10⁻⁷            s/op
QuantilesTestBenchmark.primitive:primitive·p0.95    sample                 ≈ 10⁻⁶            s/op
QuantilesTestBenchmark.primitive:primitive·p0.99    sample                  0.001            s/op
QuantilesTestBenchmark.primitive:primitive·p0.999   sample                  0.005            s/op
QuantilesTestBenchmark.primitive:primitive·p0.9999  sample                  0.013            s/op
QuantilesTestBenchmark.primitive:primitive·p1.00    sample                  0.028            s/op
```

结果发现使用`ThreadLocal`并没有提高性能，原因在于为了`flushBuffer()`，这里将`ThreadLocal`的buffer也加了锁，如果允许丟buffer中的数据，不加这把锁，应该会有更好的性能。