---
layout: post
title:  "linux性能测试与监控"
date:   2017-01-10 09:56:00
categories: linux
tags: linux test
---

* content
{:toc}

监控是架构设计中相当重要的环节，目前我的工作中涉及到监控分为系统监控与业务监控,该文主要针对线上环境（linux-debian）调试需要用到的一些监控与测试工具。





## 系统

### version

不同版本工具使用可能有所差异，首先先了解系统的基本信息：

```
uname -a #for all information regarding the kernel version,

uname -r #for the exact kernel version

lsb_release -a #for all information related to the Ubuntu version,

lsb_release -r #for the exact version

sudo fdisk -l #for partition information with all details.

lsblk -o NAME,SIZE / df -lh #All partition sizes of the HDD in Terminal
```

### load average

查看 `cat /proc/loadavg | uptime`

衡量机器负载的一个重要指标，一般影响这数值的两个因素：

 - 活跃任务
 
 - 不可中断任务（IO操作）
 
![](https://pic1.zhimg.com/v2-3a6052ede46286ffe2d00c952f817710_b.png)

所以它直接反应了系统的健康状态，它能牵引出系统诸多方面的问题

其次是数值代表的意义 `Single Core满载load值为1，那当大于1则表示有等待任务，小于则表示资源有空闲，Multiple Core满载load值则为 n*1`

## CUP

信息一般采集至/proc/vmstat
监测的工具一般iostat/top/htop/atop/dstat等等(自行发掘，大同小异)

### CPU信息

Cpu(s)表示的是cpu信息。各个值的意思是：

```
us: user cpu time (or) % CPU time spent in user space
sy: system cpu time (or) % CPU time spent in kernel space
ni: user nice cpu time (or) % CPU time spent on low priority processes
id: idle cpu time (or) % CPU time spent idle
wa: io wait cpu time (or) % CPU time spent in wait (on disk)
hi: hardware irq (or) % CPU time spent servicing/handling hardware interrupts
si: software irq (or) % CPU time spent servicing/handling software interrupts
st: steal time - - % CPU time in involuntary wait by virtual cpu while hypervisor is servicing another processor (or) % CPU time stolen from a virtual machine
```

注：st说明中范围为"窃取时间"，这是专门针对虚拟机的

### CPU性能测试

测试工具： sysbench

命令：`sysbench --test=cpu --cpu-max-prime=20000 run`

测试结果样例：

```
kylin@labu001:~$ sysbench --test=cpu --cpu-max-prime=20000 run
sysbench 0.4.12:  multi-threaded system evaluation benchmark
Running the test with following options:
Number of threads: 1
Doing CPU performance benchmark
Threads started!
Done.
Maximum prime number checked in CPU test: 20000
 
Test execution summary:
    total time:                          23.5990s
    total number of events:              10000
    total time taken by event execution: 23.5983
    per-request statistics:
         min:                                  2.34ms
         avg:                                  2.36ms
         max:                                  5.32ms
         approx.  95 percentile:               2.41ms
Threads fairness:
    events (avg/stddev):           10000.0000/0.00
    execution time (avg/stddev):   23.5983/0.00
```

## 内存

信息一般采集至/proc/meminfo
检测工具一般vmstat/free

free -m 输出的表示系统内存的使用情况：

```
              total        used        free      shared  buff/cache   available
Mem:           7902         161        6972          39         769        7414
Swap:          8109           0        8109
```

注：前面四项都比较好理解，主要说明的是 buffer/cache/available

###  buffer/cache

找不到合适的词来翻译，它们的区别在于：

A buffer is something that has yet to be "written" to disk.

A cache is something that has been "read" from the disk and stored for later use.

Buffer:用于缓冲要存储到磁盘的数据
Cache:用于缓存从磁盘读出存放到内存中待今后使用的数据。

它们的引入均是为了提供磁盘IO的性能。

### available

available 是估算可用于程序使用的可用内存，检查/proc/meminfo文件，通过将"free + Buffers/Cached"相加，预估有多少可用内存，这在十年前是可以，但是在今天肯定是不对的。

因为缓存包含存不能释放的page cache，例如shared memory segments、tmpfs、ramfs，它不包括可收回的slab内存，比如在大多数空闲的系统中，存在很多文件时，它会占用大量的系统内存。

在系统没有发生交换时，预估需要多少available内存才可以启动新的应用程序。这个available字段不同于cache或free字段所提供的数据，它除了要考虑到page cache，还要考虑到当项目在使用时，并不是所有的slabs都能被回收这一情况。

free命令中"buffers/cached"的内存，由于这块内存从操作系统的角度确实被使用，但如果用户要使用，这块内存是可以很快被回收被用户程序使用，因此从用户角度这块内存应划为空闲状态。

### 内存性能测试

#### 读性能 样本

```
sysbench --test=memory --memory-block-size=1K --memory-scope=global --memory-total-size=100G --memory-oper=read run

Running the test with following options:
Number of threads: 1
 
Doing memory operations speed test
Memory block size: 1K
 
Memory transfer size: 102400M
 
Memory operations type: read
Memory scope type: global
Threads started!
Done.
 
Operations performed: 104857600 (6045253.78 ops/sec)
 
102400.00 MB transferred (5903.57 MB/sec)
 
 
Test execution summary:
total time: 17.3454s
total number of events: 104857600
total time taken by event execution: 12.1786
per-request statistics:
min: 0.00ms
avg: 0.00ms
max: 0.06ms
approx. 95 percentile: 0.00ms
 
Threads fairness:
events (avg/stddev): 104857600.0000/0.00
execution time (avg/stddev): 12.1786/0.00
```

#### 写性能 样本

```
sysbench --test=memory --memory-block-size=1K --memory-scope=global --memory-total-size=100G --memory-oper=write run

Running the test with following options:
Number of threads: 1
 
Doing memory operations speed test
Memory block size: 1K
 
Memory transfer size: 102400M
 
Memory operations type: write
Memory scope type: global
Threads started!
Done.
 
Operations performed: 104857600 (4056443.11 ops/sec)
 
102400.00 MB transferred (3961.37 MB/sec)
 
 
Test execution summary:
total time: 25.8496s
total number of events: 104857600
total time taken by event execution: 20.6986
per-request statistics:
min: 0.00ms
avg: 0.00ms
max: 0.08ms
approx. 95 percentile: 0.00ms
 
Threads fairness:
events (avg/stddev): 104857600.0000/0.00
execution time (avg/stddev): 20.6986/0.00
```

## 磁盘
iotop

## 网络
iftop

## 参考
