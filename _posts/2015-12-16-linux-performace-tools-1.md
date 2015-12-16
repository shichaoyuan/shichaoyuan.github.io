---
layout: post
title: Linux性能工具——System Metircs
date: 2015-12-16 21:05:20
tags: performace
---


最近看到一篇文章[《Linux Performance Analysis in 60,000 Milliseconds》](http://techblog.netflix.com/2015/11/linux-performance-analysis-in-60s.html)，感觉写的很清晰明了，所以追一下作者[Brendan Gregg](http://www.brendangregg.com/)的其他文章，感觉受益匪浅，顺便做个笔记。

1. Linux性能工具——System Metircs
2. Linux性能工具——Profilers
3. ...


## 0

![Linux Performace Observability Tools](http://www.brendangregg.com/Perf/linux_observability_tools.png)

![Linux Performace Observability: sar](http://www.brendangregg.com/Perf/linux_observability_sar.png)


## uptime

```plain
$ uptime
 23:51:26 up 21:31,  1 user,  load average: 30.02, 26.43, 19.02
```

查看平均负载。

平均负载使用“等待”执行的进程数表示，三个数分别表示1分钟、5分钟、25分钟内的平均数，可以表征负载变化趋势。

上例中显示负载增加，可能意味着需要CPU，可以使用`vmstat`或`mpstat`进一步确认。

> "1 minute load average"
> 
> really means...
> > "The exponentially damped moving sum of CPU + uninterruptible disk I/O that uses a value of 60 seconds in its equation"

实际上平均负载的计算方法是指数衰减移动平均。


## dmesg | tail

```plain
$ dmesg | tail
[1880957.563150] perl invoked oom-killer: gfp_mask=0x280da, order=0, oom_score_adj=0
[...]
[1880957.563400] Out of memory: Kill process 18694 (perl) score 246 or sacrifice child
[1880957.563408] Killed process 18694 (perl) total-vm:1972392kB, anon-rss:1953348kB, file-rss:0kB
[2320864.954447] TCP: Possible SYN flooding on port 7001. Dropping request.  Check SNMP counters.
```

查看最近10条系统消息。

## vmstat 1

```plain
$ vmstat 1
procs ---------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
34  0    0 200889792  73708 591828    0    0     0     5    6   10 96  1  3  0  0
32  0    0 200889920  73708 591860    0    0     0   592 13284 4282 98  1  1  0  0
32  0    0 200890112  73708 591860    0    0     0     0 9501 2154 99  1  0  0  0
32  0    0 200889568  73712 591856    0    0     0    48 11900 2459 99  0  0  0  0
32  0    0 200890208  73712 591860    0    0     0     0 15898 4840 98  1  1  0  0
^C
```

查看服务器关键统计信息。

参数`1`表示一秒钟统计一次，第一行展示启动以来的平均值，而不是前一秒。

* **r** 在CPU上执行和等待下一轮执行的进程数。因为这个数不包括等待I/O的进程，所以相对于`load average`可以更好的表征CPU饱和状态。如果r值大于CPU核数，那么就表示饱和了。
* **free** 可用内存，单位是KB。可以用`free -m`进一步查看内存状态。
* **si,so** 交换区读写。如果不为零，表示内存不够用。
* **us,sy,id,wa,st** CPU时间。分别是用户时间，系统时间，空闲，等待I/O，stolen time。

us+sy可以用来判断CPU是否繁忙；持续的wa表示磁盘读写是一个瓶颈；sy对于I/O处理是必须的，高sy（超过20%）可能意味着内核处理I/O低效。


## mpstat -P ALL 1

```plain
$ mpstat -P ALL 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015  _x86_64_ (32 CPU)

07:38:49 PM  CPU   %usr  %nice   %sys %iowait   %irq  %soft  %steal  %guest  %gnice  %idle
07:38:50 PM  all  98.47   0.00   0.75    0.00   0.00   0.00    0.00    0.00    0.00   0.78
07:38:50 PM    0  96.04   0.00   2.97    0.00   0.00   0.00    0.00    0.00    0.00   0.99
07:38:50 PM    1  97.00   0.00   1.00    0.00   0.00   0.00    0.00    0.00    0.00   2.00
07:38:50 PM    2  98.00   0.00   1.00    0.00   0.00   0.00    0.00    0.00    0.00   1.00
07:38:50 PM    3  96.97   0.00   0.00    0.00   0.00   0.00    0.00    0.00    0.00   3.03
[...]
```

查看每个CPU的时间。

用于检查CPU资源使用不平衡，如果只有一个CPU比较忙，那么可以猜测这是单线程应用程序。

## pidstat 1

```plain
$ pidstat 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015    _x86_64_    (32 CPU)

07:41:02 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
07:41:03 PM     0         9    0.00    0.94    0.00    0.94     1  rcuos/0
07:41:03 PM     0      4214    5.66    5.66    0.00   11.32    15  mesos-slave
07:41:03 PM     0      4354    0.94    0.94    0.00    1.89     8  java
07:41:03 PM     0      6521 1596.23    1.89    0.00 1598.11    27  java
07:41:03 PM     0      6564 1571.70    7.55    0.00 1579.25    28  java
07:41:03 PM 60004     60154    0.94    4.72    0.00    5.66     9  pidstat

07:41:03 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
07:41:04 PM     0      4214    6.00    2.00    0.00    8.00    15  mesos-slave
07:41:04 PM     0      6521 1590.00    1.00    0.00 1591.00    27  java
07:41:04 PM     0      6564 1573.00   10.00    0.00 1583.00    28  java
07:41:04 PM   108      6718    1.00    0.00    0.00    1.00     0  snmp-pass
07:41:04 PM 60004     60154    1.00    4.00    0.00    5.00     9  pidstat
^C
```

查看每个进程的CPU资源占用情况。

## iostat -xz 1

```plain
$ iostat -xz 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015  _x86_64_ (32 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
          73.96    0.00    3.73    0.03    0.06   22.21

Device:   rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
xvda        0.00     0.23    0.21    0.18     4.52     2.08    34.37     0.00    9.98   13.80    5.42   2.44   0.09
xvdb        0.01     0.00    1.02    8.94   127.97   598.53   145.79     0.00    0.43    1.78    0.28   0.25   0.25
xvdc        0.01     0.00    1.02    8.86   127.79   595.94   146.50     0.00    0.45    1.82    0.30   0.27   0.26
dm-0        0.00     0.00    0.69    2.32    10.47    31.69    28.01     0.01    3.23    0.71    3.98   0.13   0.04
dm-1        0.00     0.00    0.00    0.94     0.01     3.78     8.00     0.33  345.84    0.04  346.81   0.01   0.00
dm-2        0.00     0.00    0.09    0.07     1.35     0.36    22.50     0.00    2.55    0.23    5.62   1.78   0.03
[...]
^C
```

查看块设备状态。

* **r/s,w/s,rkB/s,wkB/s** 每秒的读写次数和数据量。
* **await** 平均I/O时间（毫秒），包括排队时间和处理时间。
* **avgqu-sz** 对设备的平均请求数量。如果大于1，表示设备饱和。
* **%util** 设备利用率，显示每秒设备的工作时间。通常来说大于60%将会导致性能低下，接近100%表示设备饱和。

如果某存储设备是背后多块磁盘的逻辑盘，那么100%的利用率仅仅意味着某些I/O操作占用了全部时间，而背后的磁盘并没有饱和。

低效的磁盘I/O通常不是应用程序层面需要考虑的问题。许多异步I/O的技术可以用来改善性能，比如预读取、缓存写。

## free -m

```plain
$ free -m
             total       used       free     shared    buffers     cached
Mem:        245998      24545     221453         83         59        541
-/+ buffers/cache:      23944     222053
Swap:            0          0          0
```

查看内存状态。

* **buffers** 用于块设备I/O的buffer缓存
* **cached** 用于文件系统的page缓存

这两个数据最好不要是近乎为零，不然会导致高负载的I/O（使用`iostat`进一步确认）。

`-/+ buffers/cache`这行将buffers和cached调整为free，因为对于应用程序这些空间是可用的。

## sar -n DEV 1

```plain
$ sar -n DEV 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015     _x86_64_    (32 CPU)

12:16:48 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
12:16:49 AM      eth0  18763.00   5032.00  20686.42    478.30      0.00      0.00      0.00      0.00
12:16:49 AM        lo     14.00     14.00      1.36      1.36      0.00      0.00      0.00      0.00
12:16:49 AM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00

12:16:49 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
12:16:50 AM      eth0  19763.00   5101.00  21999.10    482.56      0.00      0.00      0.00      0.00
12:16:50 AM        lo     20.00     20.00      3.25      3.25      0.00      0.00      0.00      0.00
12:16:50 AM   docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
^C
```

查看网络接口吞吐量。

`rxkB/s`和`txkB/s`表征工作负载。上例中，eth0入量达到22MB/s，也就是176Mb/s（该网口的上限为1Gb/s）。

## sar -n TCP,ETCP 1

```plain
$ sar -n TCP,ETCP 1
Linux 3.13.0-49-generic (titanclusters-xxxxx)  07/14/2015    _x86_64_    (32 CPU)

12:17:19 AM  active/s passive/s    iseg/s    oseg/s
12:17:20 AM      1.00      0.00  10233.00  18846.00

12:17:19 AM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
12:17:20 AM      0.00      0.00      0.00      0.00      0.00

12:17:20 AM  active/s passive/s    iseg/s    oseg/s
12:17:21 AM      1.00      0.00   8359.00   6039.00

12:17:20 AM  atmptf/s  estres/s retrans/s isegerr/s   orsts/s
12:17:21 AM      0.00      0.00      0.00      0.00      0.00
^C
```

查看TCP状态。

* **active/s** 本地发起的TCP连接数（也就是通过connect())
* **passive/s** 远端发起的TCP连接数 （也就是通过accept())
* **retrans/s** TCP重传数

TCP连接数可以大致测量服务器负载，active看做outbound，passive看做inbound，但是这并不严格，因为没有考虑本地到本地的链接。

retrans可以表征网络或服务器的问题。如果这个值比较高，可能是网络不可靠，也可能是服务器负载过重而导致丢包。

## top

```
$ top
top - 00:15:40 up 21:56,  1 user,  load average: 31.09, 29.87, 29.92
Tasks: 871 total,   1 running, 868 sleeping,   0 stopped,   2 zombie
%Cpu(s): 96.8 us,  0.4 sy,  0.0 ni,  2.7 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:  25190241+total, 24921688 used, 22698073+free,    60448 buffers
KiB Swap:        0 total,        0 used,        0 free.   554208 cached Mem

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 20248 root      20   0  0.227t 0.012t  18748 S  3090  5.2  29812:58 java
  4213 root      20   0 2722544  64640  44232 S  23.5  0.0 233:35.37 mesos-slave
 66128 titancl+  20   0   24344   2332   1172 R   1.0  0.0   0:00.07 top
  5235 root      20   0 38.227g 547004  49996 S   0.7  0.2   2:02.74 java
  4299 root      20   0 20.015g 2.682g  16836 S   0.3  1.1  33:14.42 java
     1 root      20   0   33620   2920   1496 S   0.0  0.0   0:03.82 init
     2 root      20   0       0      0      0 S   0.0  0.0   0:00.02 kthreadd
     3 root      20   0       0      0      0 S   0.0  0.0   0:05.35 ksoftirqd/0
     5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
     6 root      20   0       0      0      0 S   0.0  0.0   0:06.94 kworker/u256:0
     8 root      20   0       0      0      0 S   0.0  0.0   2:38.05 rcu_sched
```

多合一工具，但是看不到随着时间的变化趋势，使用`vmstat`和`pidstat`更清楚。

另外top工具还有几个问题：

1. %CPU的计算方法各异
2. %CPU与%Cpu(s)不一致
