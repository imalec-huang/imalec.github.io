---
layout: post
title:  "Jmeter分布式测试"
date:   2018-06-07 17:19:00
categories: DevOps
tags: linux jmeter java
---

* content
{:toc}

JMeter客户端机器性能不足，模拟足够的用户来压缩服务器或受限于网络级别，则存在一个选项来控制来自单个JMeter客户端的多个远程JMeter引擎。通过远程运行JMeter，您可以跨多台低端计算机复制测试，从而模拟服务器上的更大负载。JMeter客户端的一个实例可以控制任意数量的远程JMeter实例，并从中收集所有数据。





## 执行端
默认启用ssl,如果没有配置密钥会
java.io.FileNotFoundException: rmi_keystore.jks
第一种方法：
调用bin/create-rmi-keystore.sh
创建过程最后都key输入不要设置密码
第二种方法：
jmeter.properties中ssl.disable放开注释，值设为true
启动./jmeter-server  -Djava.rmi.server.hostname=192.168.20.252

## 控制端
修改jmeter.properties添加remote.host只想slave启动的hostname，端口默认1099