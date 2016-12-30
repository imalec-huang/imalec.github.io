---
layout: post
title:  "opentsdb安装与配置说明"
date:   2016-12-29 16:19:00
categories: opentsdb
tags: opentsdb java
---

* content
{:toc}

opentsdb依赖hbase





## 准备

>set HBASE_HOME

## 下载

>wget -O opentsdb-2.2.1_all.deb https://github.com/OpenTSDB/opentsdb/releases/download/v2.2.1/opentsdb-2.2.1_all.deb

>sudo vi /etc/init.d/opentsdb 

`JDK_DIRS="/usr/local/jdk"`

>sudo vi /etc/opentsdb/logback.xml 

`:%s/\/var\/log\/opentsdb/\/data\/opentsdb\/logs/g`

>sudo vi /etc/opentsdb/opentsdb.conf

```
tsd.http.request.max_chunk = 102400
tsd.http.request.enable_chunked = true
```

## Duplicate Data Points

Writing data points in OpenTSDB is generally idempotent within an hour of the original write. This means you can write the value 42 at timestamp 1356998400 and then write 42 again for the same time and nothing bad will happen. However if you have compactions enabled to reduce storage consumption and write the same data point after the row of data has been compacted, an exception may be returned when you query over that row. If you attempt to write two different values with the same timestamp, a duplicate data point exception may be thrown during query time. This is due to a difference in encoding integers on 1, 2, 4 or 8 bytes and floating point numbers. If the first value was an integer and the second a floating point, the duplicate error will always be thrown. However if both values were floats or they were both integers that could be encoded on the same length, then the original value may be overwritten if a compaction has not occured on the row.

In most situations, if a duplicate data point is written it is usually an indication that something went wrong with the data source such as a process restarting unexpectedly or a bug in a script. OpenTSDB will fail "safe" by throwing an exception when you query over a row with one or more duplicates so you can down the issue.

With OpenTSDB 2.1 you can enable last-write-wins by setting the tsd.storage.fix_duplicates configuration value to true. With this flag enabled, at query time, the most recent value recorded will be returned instead of throwing an exception. A warning will also be written to the log file noting a duplicate was found. If compaction is also enabled, then the original compacted value will be overwritten with the latest value.

 写入OpenTSDB中的数据点是幂等的。意思是你再时间点1356998400写入值42，再写一次42，是不会有问题的。但是如果从节约存储角度考虑，写入同样的数据点需要被compacted，否则在查询这行数据的时候可能会出现异常。如果你在同一个时间点写入两个不同值，查询的时候可能会出现异常。

因为两个值可能编码类型不一样，一个是整数，一个可能是浮点数。

通常情况下，是在采集的时候进行去重。OpenTSDB在查询到重复数据的时候会返回异常，便于查找错误。

OpenTSDB2.1可以开启配置tsd.storage.fix_duplicates 。查询时候返回最近的那个条记录，而不是抛出一次。在日志中记录一条警告。

如果compatction可行，原始数据将被覆盖。