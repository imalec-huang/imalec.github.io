---
layout: post
title:  "OpenTSDB安装与配置说明"
date:   2016-12-29 16:19:00
categories: opentsdb
tags: opentsdb java
---

* content
{:toc}

OpenTSDB ，可以认为是一个时系列数据（库），它基于HBase存储数据，充分发挥了HBase的分布式列存储特性，支持数百万每秒的读写，它的特点就是容易扩展，灵活的tag机制。





## 准备

### 运行opentsdb需要满足以下条件:
- A Linux system (or Windows with manual building)
- [Java Runtime Environment 1.6 or later](https://imevis.github.io/2016/12/21/ubuntu-install-java-offline/)
- [HBase 0.92 or later](https://hbase.apache.org/book.html#quickstart)
- GnuPlot 4.2 or later(如使用opentsdb自带ui，需要安装该绘图组件)

![](http://opentsdb.net/img/tsdb-architecture.png)

## 下载安装

```
cd /tmp
wget -O opentsdb-2.3.0_all.deb https://github.com/OpenTSDB/opentsdb/releases/download/v2.3.0/opentsdb-2.3.0_all.deb
sudo dpkg --install opentsdb-2.3.0_all.deb 
```

## 配置文件

`sudo vi /etc/init.d/opentsdb`

`JDK_DIRS="/usr/local/jdk"`

do save

### 修改日志路径

创建日志目录

`sudo mkdir -p /data/opentsdb/logs`
`sudo chown -R opentsdb:opentsdb /data/opentsdb/`

`sudo vi /etc/opentsdb/logback.xml` 

替换/var/log/opentsdb 为  /data/opentsdb/logs

`:%s/\/var\/log\/opentsdb/\/data\/opentsdb\/logs/g`

do save

### 修改启动参数

`sudo vi /etc/opentsdb/opentsdb.conf`

```
tsd.http.request.max_chunk = 102400
tsd.http.request.enable_chunked = true
tsd.storage.fix_duplicates = true
tsd.storage.hbase.zk_quorum = ZK_HOST
```

## 启动查看

`sudo service opentsdb start|status|restart|stop` 


## 初始化

```
set HBASE_HOME=/opt/hbase
bash /usr/share/opentsdb/tools/create_table.sh
```


## 备注
遇到的无法启动的两个问题
 - java安装问题，导致sudo执行找到不java，可以通过sudo -s 切换到root权限下执行 java -version验证java的环境问题
 - 如果通过编译源码安装，测试opentsdb-2.3.0版本编译错误，需要将third_party包移动到build目录下，需要注意日志路径/pid文件的读写权限问题

## 部分api

`查看指标列表 http://xxx.xx.xxx.xxx:4242/api/suggest?max=1000&q=&type=metrics`
`删除指标数据 tsdb scan 2014/05/01 sum spark.ygz.sale.record  --delete`

## Duplicate Data Points

写入OpenTSDB中的数据点是幂等的。意思是你再时间点1356998400写入值42，再写一次42，是不会有问题的。但是如果从节约存储角度考虑，写入同样的数据点需要被compacted，否则在查询这行数据的时候可能会出现异常。如果你在同一个时间点写入两个不同值，查询的时候可能会出现异常。

因为两个值可能编码类型不一样，一个是整数，一个可能是浮点数。

通常情况下，是在采集的时候进行去重。OpenTSDB在查询到重复数据的时候会返回异常，便于查找错误。

OpenTSDB2.1可以开启配置tsd.storage.fix_duplicates 。查询时候返回最近的那个条记录，而不是抛出一次。在日志中记录一条警告。

如果compatction可行，原始数据将被覆盖。

Writing data points in OpenTSDB is generally idempotent within an hour of the original write. This means you can write the value 42 at timestamp 1356998400 and then write 42 again for the same time and nothing bad will happen. However if you have compactions enabled to reduce storage consumption and write the same data point after the row of data has been compacted, an exception may be returned when you query over that row. If you attempt to write two different values with the same timestamp, a duplicate data point exception may be thrown during query time. This is due to a difference in encoding integers on 1, 2, 4 or 8 bytes and floating point numbers. If the first value was an integer and the second a floating point, the duplicate error will always be thrown. However if both values were floats or they were both integers that could be encoded on the same length, then the original value may be overwritten if a compaction has not occured on the row.

In most situations, if a duplicate data point is written it is usually an indication that something went wrong with the data source such as a process restarting unexpectedly or a bug in a script. OpenTSDB will fail "safe" by throwing an exception when you query over a row with one or more duplicates so you can down the issue.

With OpenTSDB 2.1 you can enable last-write-wins by setting the tsd.storage.fix_duplicates configuration value to true. With this flag enabled, at query time, the most recent value recorded will be returned instead of throwing an exception. A warning will also be written to the log file noting a duplicate was found. If compaction is also enabled, then the original compacted value will be overwritten with the latest value.

## 参考
【[pentsdb-installation](http://opentsdb.net/docs/build/html/installation.html)】
