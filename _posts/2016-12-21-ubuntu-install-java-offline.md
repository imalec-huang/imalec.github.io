---
layout: post
title:  "Ubuntu离线安装java与证书生成"
date:   2016-12-21 20:19:00
categories: linux
tags: linux ubuntu java
---

* content
{:toc}

搭建日志服务架构的时候，因为没有使用运维自动化，每一台服务器的java环境安装不标准，导致服务总是启动异常。
整个架构体系中每台服务器都需要安装Java，和证书，下面分别贴了java安装与证书生成的脚本
（如果生产环境网络良好，可以直接使用apt安装java）





## 安装shell脚本

### 预定义证书的IP域名

```
#!/bin/bash
USER_NAME=evis
PRIVATE_TOKEN=jVraFSwVWZZX4PESyZgm
declare -A DOMAINS
DOMAINS[172.25.156.75]=es-client-01.clio.uledns.com
DOMAINS[172.25.156.77]=es-client-02.clio.uledns.com
#获取本机ip
LOCAL_IP=`ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"`
DOMAIN=${DOMAINS[$LOCAL_IP]}

if [ ! -n "${DOMAIN}" ];then
    echo "${LOCAL_IP} 找不到域名不存在"
    exit 1
fi
```

### 检验java是不是可用


```
checkJava() {
        if [ -x "$JAVA_HOME/bin/java" ]; then
                JAVA="$JAVA_HOME/bin/java"
        else
                JAVA=`which java`
        fi

        if [ ! -x "$JAVA" ]; then
                echo "Could not find any executable java binary. Please install java in your PATH or set JAVA_HOME"
                installJava
        fi
        echo "Java is installed!"
}
```

### 安装java

```
installJava() {
    JDK_HOME="/usr/lib/jvm"
    JDK_EXPORT="/etc/profile.d/oraclejdk.sh"
    
    wget -O server-jre-8u91-linux-x64.tar.gz http://172.24.138.32/software/server-jre-8u91-linux-x64.tar.gz
    tar -xf server-jre-8u91-linux-x64.tar.gz
    if [ -d $JDK_HOME ]; then
        rm -rf $JDK_HOME
    fi 
    
    mkdir $JDK_HOME
    mv jdk1.8.0_91 /usr/lib/jvm/oracle_jdk8
     
    update-alternatives --query java
    update-alternatives --query javac
    update-alternatives --install /usr/bin/java java /usr/lib/jvm/oracle_jdk8/jre/bin/java 2000
    update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/oracle_jdk8/bin/javac 2000
    update-alternatives --config java
    update-alternatives --config javac
    
    if [ -f $JDK_EXPORT ]; then 
        rm $JDK_EXPORT
    fi 
    
    echo "export J2SDKDIR=/usr/lib/jvm/oracle_jdk8" | tee -a /etc/profile.d/oraclejdk.sh
    echo "export J2REDIR=/usr/lib/jvm/oracle_jdk8/jre" | tee -a /etc/profile.d/oraclejdk.sh
    echo "export PATH=$PATH:/usr/lib/jvm/oracle_jdk8/bin:/usr/lib/jvm/oracle_jdk8/db/bin:/usr/lib/jvm/oracle_jdk8/jre/bin" | tee -a /etc/profile.d/oraclejdk.sh
    echo "export JAVA_HOME=/usr/lib/jvm/oracle_jdk8" | tee -a /etc/profile.d/oraclejdk.sh
    echo "export DERBY_HOME=/usr/lib/jvm/oracle_jdk8/db" | tee -a /etc/profile.d/oraclejdk.sh
    source /etc/profile.d/oraclejdk.sh
    echo "OK"
}
```

### 生成证书

```
generateCert() {
    SSL_PATH="/etc/pki/tls"
    KEY="${SSL_PATH}/private/${DOMAIN}.key"
    CRT="${SSL_PATH}/certs/${DOMAIN}.crt"
    
    if [ ! -f $CRT ] || [ ! -f $KEY ]; then
        echo "The cert is not exist,will be create"
        if [ ! -d $SSL_PATH ]; then
            mkdir -p /etc/pki/tls/certs
            mkdir /etc/pki/tls/private
        fi
        openssl req -subj '/CN='${DOMAIN}'/' -x509 -days 3650 -batch -nodes -newkey rsa:2048 -keyout $KEY -out $CRT
    else
        echo "The cert is exist"
    fi
}
```

### 具体安装应用（此处安装logstash，并自动回去配置文件，替换文件中的部分配置）

```
installLogstash() {
    if [ -x "/opt/logstash/bin/logstash" ]; then
        echo "logstash is installed"
    else
        wget -O logstash.deb http://172.24.138.32/software/logstash.deb
        if [ -s logstash.deb ];then
            dpkg -i logstash.deb
            #修改服务参数
            sed -i "s/#LS_HOME=\/var\/lib\/logstash/LS_HOME=\/data\/logstash/g" /etc/default/logstash
            sed -i "s/#LS_OPTS=\"\"/LS_OPTS=\"-b 8192 -w 24 -r --reload-interval 30\"/g" /etc/default/logstash
            sed -i "s/#LS_HEAP_SIZE=\"1g\"/LS_HEAP_SIZE=\"10g\"/g" /etc/default/logstash
            sed -i "s/#LS_LOG_FILE=\/var\/log\/logstash\/logstash.log/LS_LOG_FILE=\/data\/logstash\/logs\/logstash.log/g" /etc/default/logstash
            sed -i "s/#LS_USE_GC_LOGGING=\"true\"/LS_USE_GC_LOGGING=\"true\"/g" /etc/default/logstash
            sed -i "s/#LS_GC_LOG_FILE=\/var\/log\/logstash\/gc.log/LS_GC_LOG_FILE=\/data\/logstash\/logs\/gc.log/g" /etc/default/logstash
            sed -i "s/#LS_CONF_DIR=\/etc\/logstash\/conf.d/LS_CONF_DIR=\/etc\/logstash\/conf.d/g" /etc/default/logstash
            mkdir -p /data/logstash/logs
            mkdir /etc/logstash/templates
            chown -R logstash:logstash /data/logstash
        else
            echo "Not exist install package"
        fi
    fi
    #下载配置文件
    wget -O 100-beat-input.conf git.uletm.com/$USER_NAME/elk-script/raw/master/logstash/conf.d/100-beat-input.conf?private_token=$PRIVATE_TOKEN
    wget -O 200-initialize-filter.conf git.uletm.com/$USER_NAME/elk-script/raw/master/logstash/conf.d/200-initialize-filter.conf?private_token=$PRIVATE_TOKEN
    wget -O 210-webaccess-filter.conf git.uletm.com/$USER_NAME/elk-script/raw/master/logstash/conf.d/210-webaccess-filter.conf?private_token=$PRIVATE_TOKEN
    wget -O 211-audit-filter.conf git.uletm.com/$USER_NAME/elk-script/raw/master/logstash/conf.d/211-audit-filter.conf?private_token=$PRIVATE_TOKEN
    #wget -O 212-track-filter.conf git.uletm.com/$USER_NAME/elk-script/raw/master/logstash/conf.d/212-track-filter.conf?private_token=$PRIVATE_TOKEN
    wget -O 299-common-filter.conf git.uletm.com/$USER_NAME/elk-script/raw/master/logstash/conf.d/299-common-filter.conf?private_token=$PRIVATE_TOKEN
    wget -O 300-elasticsearch-output.conf git.uletm.com/$USER_NAME/elk-script/raw/master/logstash/conf.d/300-elasticsearch-output.conf?private_token=$PRIVATE_TOKEN
    mv *.conf /etc/logstash/conf.d
    sed -i "s/#domain#/${DOMAIN}/g" /etc/logstash/conf.d/100-beat-input.conf
    sed -i "s/20480/8192/g" /etc/default/logstash
    sed -i "s/10g/8g/g" /etc/default/logstash
}
```

### 执行

```
#第一步 自更新安装脚本
wget -O installLS.sh git.uletm.com/$USER_NAME/elk-script/raw/master/logstash/installLS.sh?private_token=$PRIVATE_TOKEN
#第二步 检测是否安装java，如果未安装将安装java
checkJava
#第三步 生成机器证书
generateCert
#第四步 安装配置logstash
installLogstash
#第五步 启动logstash
service logstash restart
```