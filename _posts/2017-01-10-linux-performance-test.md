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

属于一般采集至/proc/vmstat,单个的状态查看/proc/pid/xxx,监测的工具一般iostat/top/dstat等等(自行发掘，大同小异)

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

## 磁盘

## 网络

## 参考
