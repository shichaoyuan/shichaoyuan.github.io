---
layout: post
title:  "命令logrotate"
date:   2015-01-29 19:11:14
tags: log
---

# 1

最近在做的项目中有个小问题：

nginx的log不支持自动分割（类似log4j的`DailyRollingFileAppender`）。

查了一下，有个小命令logrotate可以很好的解决这个问题。

# 2

logrotate在笔者使用的服务器`CentOS release 6.5`上已经自带安装了，所以略过安装的部分。

首先，在`/etc/logrotate.d/nginx`中配置log文件的分割逻辑，

```sh
/home/xxx/nginx/logs/*.log {
    daily
    dateext
    compress
    rotate 30
    create 644 xxx xxx
    delaycompress
    notifempty
    sharedscripts
    postrotate
        [ -f /usr/local/nginx/logs/nginx.pid ] && kill -USR1 `cat /usr/local/nginx/logs/nginx.pid`
    endscript
}
```

具体的参数详见[logrotate8.html](http://linuxcommand.org/man_pages/logrotate8.html)

需要解释一下的是delaycompress和sharedscripts这两个参数。

> delaycompress

> Postpone  compression of the previous log file to the next rotation cycle. This has only effect when used in combination with compress. It can be used when some program can not be told to close its logfile and thus might continue writing to the previous log file for some time.

简而言之，20号执行命令，那么压缩19号的log文件。
如果马上压缩，那么就会使得分割之后（kill -USR1之前）的部分log丢失，如果隔一天再压缩就没有这个问题。

> sharedscripts

> Normally, prescript and postscript scripts are run for each log which is rotated, meaning that a single script may be run multiple  times for log file entries which match multiple files (such as the /var/log/news/* example). If sharedscript is specified, the scripts are only run once, no matter how many logs match the wildcarded pattern. However, if none of the logs in the pattern require  rotating,  the  scripts  will  not  be run at all. This option overrides the nosharedscripts option and implies create option.

因为用到了通配符，sharedscripts的作用就是在所有log文件都处理完后再执行下面的脚本，如果没有添加这个参数，那么每处理完一个文件就执行一次脚本。


# 3

好的，配置完了，那么logrotate是怎么执行的呢？

很简单，基于CRON。

`/etc/logrotate.conf`是logrotate的配置文件，前面nginx中配置的参数，会覆盖其中的缺省参数。

接着往上找，`/etc/cron.daily/logrotate`

```sh
#!/bin/sh

/usr/sbin/logrotate /etc/logrotate.conf >/dev/null 2>&1
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
```

这个脚本很简单，就是执行logrotate这个命令。

再接着往上找，`/etc/cron.d/0daily`

```sh
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
HOME=/
5 1 * * * root run-parts /etc/cron.daily
```

这里指定了`cron.daily`的执行时间。

#4

如果不想等，想直接看效果，怎么办？

执行`logrotate -f /etc/logrotate.d/nginx`。

为什么执行第一次有效果，执行第二次就没有效果了？

因为logrotate会记录执行的状态，默认保存在`/var/lib/logrotate.status`中。

``` plain
logrotate state -- version 2
"/home/xxx/nginx/logs/error.log" 2015-1-29
"/home/xxx/nginx/logs/access.log" 2015-1-29
"/home/xxx/nginx/logs/*.log" 2015-1-12
```

如果想再次执行，那么修改一下日期，或者删掉改行就行。

#5

还有一个比较容易忽略的问题，

`/etc/logrotate.d/nginx`文件的mode建议为`644`，否则可能无法执行。

具体的原因参见[Changeset 302](https://fedorahosted.org/logrotate/changeset/302)

