---
layout: post
title:  "Jvm远程连接监控调试"
date:   2017-01-11 09:56:00
categories: java
tags: java linux jvm debug
---

* content
{:toc}

生产环境下往往出现诸多问题，为了更精准定位问题需要，往往需要线上调试，java应用最直接的方式莫过于对jvm监控，包括通过分析cpu/heap分布/线程等分析问题。官方提供两种方式远程监控jvm，基于rmi技术的jstatd与配置jmx代理方式，如果拥有服务器权限还有第三种方式，使用jdk提供的工具命令查看





## [RMI-Jstatd](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstatd.html)

jstatd命令是一个RMI服务器应用程序，用于监视运行的Java HotSpot VM，并提供一个接口，以使远程监视工具能够绑定到在本地主机上运行的JVM

这种方式的优点是不需要重启应用做额外配置，但是也存在缺点，就是信息不够全面，测试下无法获取cpu信息已经线程的详细信息

创建策略文件
`sudo vi /home/user/jstatd.all.policy`

内容如下：（手动替换变量）

```
grant codebase "file:${java.home}/lib/tools.jar" {
   permission java.security.AllPermission;
};
```

启动命令：（手动替换变量）

```
${JAVA_HOME}/bin/jstatd
 -J-Djava.security.policy=/home/user/jstatd.all.policy
 -J-Djava.rmi.server.hostname=${hostname}
 -J-Djava.rmi.server.logCalls=true
```

## [JMX-Option](http://docs.oracle.com/javase/8/docs/technotes/guides/management/agent.html)

添加启动参数 JAVA_OPTS，内容如下：（无权限控制，生产注意安全）

```
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=2099
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

## [BIN-Troubleshooting](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/s11-troubleshooting_tools.html)

常用命令如下：

### [查看线程状态](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstack.html)

`jstack vmid`

### [查看内存存活对象/heap分布](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jmap.html)

`jmap -histo:live vmid`

`jmap -heap vmid`

### [查看GC状态](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)

`jstat -gc|gcutil vmid -ms -count`

sudo环境下未配置JAVA_HOME，使用绝对路径

`sudo JAVA_HOE/bin/jxxx -options vmid`

测试中部分命令需要程序启动用户运行(histo:live)，如下：

`sudo -u user JAVA_HOMT/bin/jxxx -options vmid`

## linux 查看线程数
- top -H : Threads toggle 加上这个选项启动top，top一行显示一个线程。否则，它一行显示一个进程。

- ps xH：H Show threads as if they were processes 这样可以查看所有存在的线程。

- ps -mp PID：m Show threads after processes 这样可以查看一个进程起的线程数。

- cat /proc/PID/status 进程的详细信息