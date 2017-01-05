---
layout: post
title:  "Ubuntu 16.04中安装Apache Tomcat 8服务"
date:   2017-01-03 09:56:00
categories: tomcat
tags: tomcat8 java ubuntu
---

* content
{:toc}

Apache Tomcat是用于服务的Java应用程序的Web服务器和servlet容器。Tomcat是一个开放源代码实现的Java Servlet和JavaServer Pages技术，由Apache软件基金会发布。本教程介绍了基本安装和你的Ubuntu 16.04服务器上的Tomcat 8的最新版本的一些配置。






## 准备

开始之前，你应该有一个具有sudo操作权限的非root用户，jdk安装参考[Ubuntu离线安装Java](https://imevis.github.io/2016/12/21/ubuntu-install-java-offline/)。

![](https://community-cdn-digitalocean-com.global.ssl.fastly.net/assets/tutorials/images/large/tomcat_tutorial_tw_pat.png?1468953231)

## Step1 创建tomcat用户

出于安全的考虑，tomcat应该使用一个非root权限的用户运行，所以我们将创建一个tomcat用户组和用户来启动tomcat服务
首先我们创建一个group:

`$ sudo groupadd tomcat`

创建一个新的 tomcat用户。我们将使该用户的成员tomcat组，一个主目录/opt/tools/tomcat（这里我们将安装Tomcat），并将shell login 设置为 /bin/false（所以没有人可以登录到帐户）

`$ sudo useradd -s /bin/false -g tomcat -d /opt/tools/tomcat tomcat`

## Step2 下载安装tomcat

切换至临时目录下载tomcat压缩包

```
cd /tmp
curl -O http://ftp.yz.yamagata-u.ac.jp/pub/network/apache/tomcat/tomcat-8/v8.5.9/bin/apache-tomcat-8.5.9.tar.gz
```

创建目录，解压tomcat压缩包

```
sudo mkdir /opt/tools/tomcat
sudo tar xzvf apache-tomcat-8*tar.gz -C /opt/tools/tomcat --strip-components=1
```

## Step3 修改权限

```
cd /opt/tools/tomcat
sudo chmod a+w /opt/tools
sudo chgrp -R tomcat /opt/tools/tomcat
sudo chmod -R g+r conf
sudo chmod g+x conf
sudo chown -R tomcat webapps/ work/ temp/ logs/
```

## Step4 创建systemd服务文件

 获取java安装路径

`sudo update-java-alternatives -l`

 输出

`java-1.8.0-openjdk-amd64       1081       /usr/lib/jvm/oracle_jdk8`

JAVA_HOME

`/usr/lib/jvm/oracle_jdk8/jre`

创建service文件

`sudo vi /etc/systemd/system/tomcat.service`

```
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking

Environment=JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre
Environment=CATALINA_PID=/opt/tools/tomcat/temp/tomcat.pid
Environment=CATALINA_HOME=/opt/tools/tomcat
Environment=CATALINA_BASE=/opt/tools/tomcat
Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'
Environment='JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom'

ExecStart=/opt/tools/tomcat/bin/startup.sh
ExecStop=/opt/tools/tomcat/bin/shutdown.sh

User=tomcat
Group=tomcat
UMask=0007
RestartSec=10
Restart=always

[Install]
WantedBy=multi-user.target
```

## Step5 配置tomcat管理用户

`sudo vi /opt/tools/tomcat/conf/tomcat-users.xml`

```
<role rolename="manager"/>
<role rolename="manager-gui"/>
<role rolename="admin"/>
<role rolename="admin-gui"/>
<user username="tomcat" password="tomcat" roles="admin-gui,admin,manager-gui,manager"/>
```

修改配置

`sudo vi /opt/tools/tomcat/webapps/manager/META-INF/context.xml`

和

`sudo vi /opt/tools/tomcat/webapps/host-manager/META-INF/context.xml`

```
context.xml files for Tomcat webapps
<Context antiResourceLocking="false" privileged="true" >
  <!--<Valve className="org.apache.catalina.valves.RemoteAddrValve"
         allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />-->
</Context>
```

## Step6 启动查看tomcat

`sudo systemctl daemon-reload`

`sudo systemctl start tomcat`

`sudo systemctl status tomcat`

### 重启tomcat
 
`sudo systemctl restart tomcat`

### 访问地址

> http://remote_address:8080

## 参考
【[How To Install Apache Tomcat 8 on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-install-apache-tomcat-8-on-ubuntu-16-04)】