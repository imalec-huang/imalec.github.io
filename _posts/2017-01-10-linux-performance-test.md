---
layout: post
title:  "linux性能测试与监控"
date:   2017-01-10 09:56:00
categories: Linux
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
us: user cpu time (or) % CPU time spent in user space 用户空间上的
sy: system cpu time (or) % CPU time spent in kernel space 用在系统空间上
ni: user nice cpu time (or) % CPU time spent on low priority processes
id: idle cpu time (or) % CPU time spent idle 空闲
wa: io wait cpu time (or) % CPU time spent in wait (on disk) 用在等待磁盘IO
hi: hardware irq (or) % CPU time spent servicing/handling hardware interrupts 硬中断
si: software irq (or) % CPU time spent servicing/handling software interrupts 软中断
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

Buffer:预写
Cache:预读。

它们的引入均是为了提供磁盘IO的性能。

### available

available 是估算可用于程序使用的可用内存，检查/proc/meminfo文件，通过将"free + Buffers/Cached"相加，预估有多少可用内存，这在十年前是可以，但是在今天肯定是不对的。

因为缓存包含存不能释放的page cache，例如shared memory segments、tmpfs、ramfs，它不包括可收回的slab内存，比如在大多数空闲的系统中，存在很多文件时，它会占用大量的系统内存。

在系统没有发生交换时，预估需要多少available内存才可以启动新的应用程序。这个available字段不同于cache或free字段所提供的数据，它除了要考虑到page cache，还要考虑到当项目在使用时，并不是所有的slabs都能被回收这一情况。

free命令中"buffers/cached"的内存，由于这块内存从操作系统的角度确实被使用，但如果用户要使用，这块内存是可以很快被回收被用户程序使用，因此从用户角度这块内存应划为空闲状态。

### 内存性能测试

#### 读性能 样本

```
sysbench --test=memory --memory-block-size=4K --memory-scope=global --memory-total-size=100G --memory-oper=read run

Running the test with following options:
Number of threads: 1
 
Doing memory operations speed test
Memory block size: 4K
 
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
sysbench --test=memory --memory-block-size=4K --memory-scope=global --memory-total-size=100G --memory-oper=write run

Running the test with following options:
Number of threads: 1
 
Doing memory operations speed test
Memory block size: 4K
 
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

### 信息

磁盘一般是系统中最容易出现性能瓶颈的地方，首先了解下磁盘IO的工作基本知识：

`/usr/bin/time -v date`

1.对于文件的读写不会直接对磁盘读写（buffers/cache）

2.磁盘与内存之间交互通过页(Page size (bytes) 4096)

3.页的两个管理进程kswapd 和 pdflush。

4.两种缺页中断动作

 - 主缺页中断-从磁盘读取缺页(Major page faults)
 
 - 次缺页中断-从缓存读取缺页(Minor page faults)

5.页的三种类型

 - Read pages，只读页（或代码页）- 不可修改页。
  
 - Dirty pages，脏页 - 内存中发生修改，待同步到磁盘文件中的页。 
 
 - Anonymous pages，匿名页  - 无文件关联页。 

6.IOPS - 磁盘的每秒可访问次数

7.KB per IO  =  每秒读写 IO 字节数除以每秒读写 IOPS 数，rkB/s 除以 r/s，wkB/s 除以 w/s

8.顺序 IO 和 随机 IO 

### 监控

`iostat -kx 1`

DISK属性值说明：

```
rrqm/s:  每秒进行 merge 的读操作数目。即 rmerge/s
wrqm/s:  每秒进行 merge 的写操作数目。即 wmerge/s
r/s:  每秒完成的读 I/O 设备次数。即 rio/s
w/s:  每秒完成的写 I/O 设备次数。即 wio/s
rsec/s:  每秒读扇区数。即 rsect/s
wsec/s:  每秒写扇区数。即 wsect/s
rkB/s:  每秒读K字节数。是 rsect/s 的一半，因为每扇区大小为512字节。
wkB/s:  每秒写K字节数。是 wsect/s 的一半。
avgrq-sz:  平均每次设备I/O操作的数据大小 (扇区)。
avgqu-sz:  平均I/O队列长度。
await:  平均每次设备I/O操作的等待时间 (毫秒)。
svctm: 平均每次设备I/O操作的服务时间 (毫秒)。
%util:  一秒中有百分之多少的时间用于 I/O 操作，即被io消耗的cpu百分比
```

备注：如果 %util 接近 100%，说明产生的I/O请求太多，I/O系统已经满负荷，该磁盘可能存在瓶颈。如果 svctm 比较接近 await，说明 I/O 几乎没有等待时间；如果 await 远大于 svctm，说明I/O 队列太长，io响应太慢，则需要进行必要优化。如果avgqu-sz比较大，也表示有当量io在等待。

### 性能

测试样本

```
sysbench --test=fileio --file-total-size=1G prepare

sysbench --test=fileio --file-total-size=1G --file-test-mode=rndrw --init-rng=on --max-time=300 --max-requests=0 run

Running the test with following options:
Number of threads: 1
Initializing random number generator from timer.
 
 
Extra file open flags: 0
128 files, 8Mb each
1Gb total file size
Block size 16Kb
Number of random requests for random IO: 0
Read/Write ratio for combined random IO test: 1.50
Periodic FSYNC enabled, calling fsync() each 100 requests.
Calling fsync() at the end of test, Enabled.
Using synchronous I/O mode
Doing random r/w test
Threads started!
Time limit exceeded, exiting...
Done.
 
Operations performed: 33000 Read, 22000 Write, 70340 Other = 125340 Total
Read 515.62Mb Written 343.75Mb Total transferred 859.38Mb (2.8644Mb/sec)
183.32 Requests/sec executed
 
Test execution summary:
total time: 300.0153s
total number of events: 55000
total time taken by event execution: 0.4013
per-request statistics:
min: 0.00ms
avg: 0.01ms
max: 0.10ms
approx. 95 percentile: 0.01ms
 
Threads fairness:
events (avg/stddev): 55000.0000/0.00
execution time (avg/stddev): 0.4013/0.00
```

备注：--file-test-mode   文件测试模式，包含seqwr（顺序写）、seqrewr（顺序读写）、seqrd（顺序读）、rndrd（随即读）、rndwr（随机写）、rndrw（随机读写）,liunx内存交换数据大小以页为单位，一页大小4k，测试block设置为4k

## 网络

`网络测试监控极为复杂，详细方法参考【参考】博文`

这里只关注流量监控
iftop/atop/nload -m

## 参考

【[Linux按照CPU、内存、磁盘IO、网络性能监测](https://blog.csdn.net/huangjin0507/article/details/51879705)】
