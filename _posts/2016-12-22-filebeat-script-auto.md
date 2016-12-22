---
layout: post
title:  "日志收集agent-Filebeat自动化脚本"
date:   2016-12-22 21:19:00
categories: elastic
tags: beats filebeat go
---

* content
{:toc}

日志服务架构中关于对日志收集技术选型主要考虑：
1.轻量级
2.断点续传
3.横向拓展
filebeat基于go语言，新版本5.×支持输出kafka
本文提供filebeat自动化安装运行脚本与配置自动生产脚本（最初脚本使用在Apache配置文件自动提取为配置文件，抓取apache日志）





## filebeat 自动化脚本

>脚本只提供一个参考样本，实际运用根据自身环境修改其中参数和逻辑

### 自动运维shell脚本

```
#!/bin/bash
FILEBEAT_HOME=/opt/filebeat
USER_NAME=evis
PRIVATE_TOKEN=jVraFSwVWZZX4PESyZgm
IP_ADDRESS=$2
CONTAINER_NAME=$3

#启动方法
start(){
    if [ "$CONTAINER_NAME" = "all" ];then
        for i in `ls $FILEBEAT_HOME/etc|sed "s/.yml//g"`
        do
            /sbin/start-stop-daemon -S -b -m -p $FILEBEAT_HOME/run/${i}.pid --exec $FILEBEAT_HOME/bin/filebeat -- -c $FILEBEAT_HOME/etc/${i}.yml
            if [ 0 -eq $? ];then
                echo "${i} running"
            fi
        done
    else
        CONTAINER_NAME=`echo $CONTAINER_NAME|sed 's/tomcat/tmct/g'|sed 's/jboss/jbs/g'|sed 's/timer/tmr/g'`
        /sbin/start-stop-daemon -S -b -m -p $FILEBEAT_HOME/run/filebeat-${CONTAINER_NAME}.pid --exec $FILEBEAT_HOME/bin/filebeat -- -c $FILEBEAT_HOME/etc/filebeat-${CONTAINER_NAME}.yml
        if [ 0 -eq $? ];then
            echo "filebeat-${CONTAINER_NAME} running"
        fi
    fi
}
#停止
stop(){
    if [ "$CONTAINER_NAME" = "all" ];then
        for i in `ls $FILEBEAT_HOME/etc|sed "s/.yml//g"`
        do
            /sbin/start-stop-daemon -K -p $FILEBEAT_HOME/run/${i}.pid
            if [ 0 -eq $? ];then
                echo "${i} is stoped"
            fi
            rm -f $FILEBEAT_HOME/run/${i}.pid
        done
    else
        CONTAINER_NAME=`echo $CONTAINER_NAME|sed 's/tomcat/tmct/g'|sed 's/jboss/jbs/g'|sed 's/timer/tmr/g'`
        /sbin/start-stop-daemon -K -p $FILEBEAT_HOME/run/filebeat-${CONTAINER_NAME}.pid
        if [ 0 -eq $? ];then
            echo "filebeat-${CONTAINER_NAME} is stoped"
        fi
        rm -f $FILEBEAT_HOME/run/filebeat-${CONTAINER_NAME}.pid
    fi
}
#状态
status(){
    if [ "$CONTAINER_NAME" = "all" ];then
        for i in `ls $FILEBEAT_HOME/etc|sed "s/.yml//g"`
        do
            /sbin/start-stop-daemon -T -p $FILEBEAT_HOME/run/${i}.pid
            if [ 0 -eq $? ];then
                echo "${i} is running"
            fi
        done
    else
        CONTAINER_NAME=`echo $CONTAINER_NAME|sed 's/tomcat/tmct/g'|sed 's/jboss/jbs/g'|sed 's/timer/tmr/g'`
        /sbin/start-stop-daemon -T -p $FILEBEAT_HOME/run/filebeat-${CONTAINER_NAME}.pid
        if [ 0 -eq $? ];then
            echo "filebeat-${CONTAINER_NAME} is running"
        fi
    fi
}
#安装或者更新
install(){
    egrep "^zabbix" /etc/group >& /dev/null  
    if [ $? -ne 0 ];then  
        echo "Not Found group - zabbix"
        exit 1
    fi  
      
    egrep "^zabbix" /etc/passwd >& /dev/null  
    if [ $? -ne 0 ];then
        echo "Not Found user - zabbix"
        exit 1
    fi
    if [ ! -d "$FILEBEAT_HOME/bin" ]; then #如果不存在配置文件目录
        mkdir -p $FILEBEAT_HOME/bin
    fi
    if [ ! -d "$FILEBEAT_HOME/etc" ]; then #如果不存在配置文件目录
        mkdir $FILEBEAT_HOME/etc
    fi
    if [ ! -d "$FILEBEAT_HOME/run" ];then #如果不存在运行时目录
        mkdir $FILEBEAT_HOME/run
    fi
    if [ ! -d "/data/logs/filebeat" ];then #如果不存在运行时目录
        mkdir /data/logs/filebeat
    fi
    wget -O filebeat.tar.gz http://172.24.138.32/software/filebeat.tar.gz
    tar -xvf filebeat.tar.gz
    mv ./filebeat*/filebeat $FILEBEAT_HOME/bin
    chown -R zabbix:zabbix $FILEBEAT_HOME
    chown -R /data/logs/filebeat
    rm -rf filebeat.tar.gz filebeat-*
}
#日志配置方法
config(){
    if [ "$CONTAINER_NAME" = "all" ];then
        #获取配置脚本tar包
        wget -O filebeat.tar.gz git.uletm.com/$USER_NAME/elk-script/raw/master/filebeat/tar/filebeat-${IP_ADDRESS}.tar.gz?private_token=$PRIVATE_TOKEN
        #如果脚本不为空
        if [ -s filebeat.tar.gz ];then
            tar -xvf filebeat.tar.gz
            mv ./configs/$IP_ADDRESS/filebeat-*.yml $FILEBEAT_HOME/etc
            rm -rf configs
        else
            echo "Get filebeat.tar.gz failure"
            exit 1
        fi
    else
        #获取配置脚本
        wget -O generator_filebeat.py git.uletm.com/$USER_NAME/elk-script/raw/master/filebeat/generator_filebeat.py?private_token=$PRIVATE_TOKEN
        #如果脚本不为空
        if [ -s generator_filebeat.py ];then
            python generator_filebeat.py $IP_ADDRESS $CONTAINER_NAME $FILEBEAT_HOME/etc/
        else
            echo "Get generator_filebeat.py failure"
            exit 1
        fi
    fi
}
case "$1" in
    config)
        if [ ! -n "$IP_ADDRESS" ] || [ ! -n "$CONTAINER_NAME" ];then
            echo "Missing arguments! "
            exit 1
        fi
        if [ ! -f "$FILEBEAT_HOME/bin/filebeat" ];then #如果没有安装
            echo "Filebeat is not installed, will be install!"
            exit 1
        fi
        config
    ;;
    restart)
        stop
        sleep 1
        start
    ;;
    update)
        config
        sleep 1
        stop
        sleep 1
        start
    ;;
    start)
        start
        ;;
    stop)
        stop
    ;;
    status)
        status
    ;;
    install)
        install
    ;;
    upgrade)
        wget -O filebeat.sh git.uletm.com/$USER_NAME/elk-script/raw/master/filebeat/filebeat.sh?private_token=$PRIVATE_TOKEN
        if [ -s filebeat.sh ];then
            echo "filebeat.sh is OK"
        fi
    ;;
    *)
        echo "Usage: start <localIP> - 启动服务器指定IP上所有的agent"
        echo "       restart <localIP> - 重启服务器指定所有的agent"
        echo "       config <localIP> - 业务服务器日志agent自动配置"
        echo "       apache <localIP> - apache服务器日志agent自动配置"
        echo "       stop <localIP> - 停止所有的agent"
        echo "       status <localIP> -查看所有的agent启动状态"
        echo "       install - 安装或更新filebeat"
        echo "       upgrade - 脚本自更新"
        exit 1
    ;;
esac
exit 0
```

## filebeat配置自动生成脚本

>自动提取方法的实现根据自身数据格式修改，脚本主要提供模板参考

### Apache配置文件样本

```
<VirtualHost *:80>
    DocumentRoot "/data/postmall"
    ServerName money.ule.com
    JkMount /lendcs/* lendcs
    JkMount /lendvps/* lendvps
    JkMount /ifadmin/* ifadmin
    JkMount /lendMerchant/*  lendmerchant
    JkMount /uhjservice/* uhjservice
    JkMount /uhjWelabAuditApi/* uhjWelabAuditApi
    ErrorLog "/data/logs/apache/uhj-error.log"
    CustomLog "|/usr/sbin/rotatelogs /data/logs/apache/uhj-access.%Y%m%d.log 86400 480" apache_access_format
</VirtualHost>
```

### 自动配置文件生成Python脚本

```
# -*- coding: utf-8 -*-
import codecs
import httplib
import json
import re
import sys
reload(sys)
sys.setdefaultencoding("utf-8")  # @UndefinedVariable
class Builder():
    __serverHosts = ["logstash-01.clio.uledns.com",
                  "logstash-02.clio.uledns.com",
                  "logstash-03.clio.uledns.com",
                  "logstash-04.clio.uledns.com",
                  "logstash-05.clio.uledns.com",
                  "logstash-06.clio.uledns.com",
                  "logstash-07.clio.uledns.com",
                  "logstash-08.clio.uledns.com",
                  "logstash-09.clio.uledns.com",
                  "logstash-10.clio.uledns.com",
                  "logstash-11.clio.uledns.com",
                  "logstash-12.clio.uledns.com"]
    __serverPorts = [5044, 5045, 5046, 5047, 5048]
    __connections = {}
    __logs = []  # log config list
    __fields = []
    __target = ''
    __hashcode = 0

    # 定义构造方法
    def __init__(self, applogs, target, container, hashcode):
        self.__logs = applogs
        self.__target = target
        self.__fields = applogs[0].keys()
        self.__container = container
        self.__hashcode = hashcode

    # FileBeat配置文件
    def buildFileBeat(self):
        self.__fields.remove("path")
        outFile = codecs.open(self.__target + 'filebeat-' + self.__container + '.yml', 'wb+', 'utf-8')
        print >> outFile, ('shipper:')
        print >> outFile, ('  name: %s' % (self.__container))
        print >> outFile, ('  queue_size: 1000')
        print >> outFile, ('filebeat:')
        print >> outFile, ('  spool_size: 2048')
        print >> outFile, ('  idle_timeout: 2s')
        print >> outFile, ('  registry_file: /data/logs/filebeat/filebeat-%s/.filebeat' % (self.__container))
        print >> outFile, ('  publish_async: false')
        print >> outFile, ('  prospectors:')
        for result in self.__logs:
            print >> outFile, ('  - paths:')
            print >> outFile, ('    - %s' % (result["path"]))
            print >> outFile, ('    document_type: logs')
            print >> outFile, ('    ignore_older: 24h')
            print >> outFile, ('    close_older: 1h')
            print >> outFile, ('    scan_frequency: 10s')
            print >> outFile, ('    tail_files: true')
            print >> outFile, ('    force_close_files: true')
            print >> outFile, ('    fields_under_root: true')
            print >> outFile, ('    fields:')
            for field in self.__fields:
                print >> outFile, ('      %s: %s' % (field, result[field]))
            if self.__container != 'pch':
                print >> outFile, ('    multiline:')
                print >> outFile, ('      pattern: \'^[[:space:]]+|^Caused by:\'')
                print >> outFile, ('      negate: false')
                print >> outFile, ('      match: after')
                print >> outFile, ('      max_lines: 500')
        print >> outFile, ('output:')
        print >> outFile, ('  logstash:')
        print >> outFile, ('    hosts:')
        print >> outFile, ('    - %s:%s' % (self.__serverHosts[self.__hashcode % len(self.__serverHosts)], self.__serverPorts[self.__hashcode % len(self.__serverPorts)]))
        print >> outFile, ('    - %s:%s' % (self.__serverHosts[(self.__hashcode + 2) % len(self.__serverHosts)], self.__serverPorts[self.__hashcode % len(self.__serverPorts)]))
        print >> outFile, ('    - %s:%s' % (self.__serverHosts[(self.__hashcode + 4) % len(self.__serverHosts)], self.__serverPorts[self.__hashcode % len(self.__serverPorts)]))
        print >> outFile, ('    loadbalance: true')
        print >> outFile, ('    index: ule-%s' % ('business' if self.__container != 'pch' else "webaccess"))
        print >> outFile, ('    tls:')
        print >> outFile, ('      certificate_authorities:')
        print >> outFile, ('      - /etc/pki/tls/certs/%s.crt' % (self.__serverHosts[self.__hashcode % len(self.__serverHosts)]))
        print >> outFile, ('      - /etc/pki/tls/certs/%s.crt' % (self.__serverHosts[(self.__hashcode + 2) % len(self.__serverHosts)]))
        print >> outFile, ('      - /etc/pki/tls/certs/%s.crt' % (self.__serverHosts[(self.__hashcode + 4) % len(self.__serverHosts)]))
        print >> outFile, ('logging:')
        print >> outFile, ('  level: info')
        print >> outFile, ('  to_files: true')
        print >> outFile, ('  to_syslog: false')
        print >> outFile, ('  files:')
        print >> outFile, ('    path: /data/logs/filebeat/filebeat-%s' % (self.__container))
        print >> outFile, ('    rotateeverybytes: 104857600')
        print >> outFile, ('    name: filebeat.log')
        print >> outFile, ('    keepfiles: 7')
        outFile.close()
        
class Extracter:
    # 定义构造方法
    def __init__(self, hostip, containerName):
        self.__hostip = hostip
        self.__containerName = containerName
    def extractApplicationConf(self):
        httpClient = None
        responseResult = None
        try:
            httpClient = httplib.HTTPConnection('gr.uletm.com', 80, timeout=30)
            httpClient.request('GET', '/sam/api/clioI?ip=' + self.__hostip + '&container=' + self.__containerName)
            response = httpClient.getresponse()
            print response.status, response.reason
            print "Start read response ..."
            responseResult = response.read()
        except Exception, e:
            print e
        finally:
            if httpClient:
                httpClient.close()
                print "Response Over!!!"
        if responseResult:
            applogs = []
            results = json.loads(responseResult, "utf-8")
            if len(results) == 0:
                print "【%s - %s】 Can't found item!!!" % (hostip, containerName)
                exit()
            for result in results:
                if "profile" in result and "logs" in result["profile"]:
                    container = (result["server"] if "server" in result else "container")
                    module = (result["module"] if "module" in result else "")
                    app = (result["app"] if "app" in result else "")
                    assigner = (result["assigner"] if "assigner" in result else "")
                    for log in result["profile"]["logs"]:
                        if not (re.match(r'[\w/-_.]', log["path"])):
                            print log["path"]
                        config = {"module": module, "app": app, "container": container, "assigner": assigner, "hostip": self.__hostip,
                                  "logid": (log["id"] if "id" in log else ""),
                                  "logname": (log["name"] if "name" in log else ""),
                                      "path": (log["path"] if "path" in log else ""), "ver":1}
                        applogs.append(config)
        return applogs
    
    def extractApacheConf(self):
        pattern = re.compile(r'(^ServerName\s*)|(^CustomLog\s*)')
        config = open("/usr/local/apache/conf/extra/httpd-vhosts.conf")
        applogs = []
        while 1:
            result = ''
            lines = config.readlines()
            if not lines:
                break
            for line in lines:
                line = line.strip()
                if "<VirtualHost *:80>" in line:
                    result = ''
                elif ("</VirtualHost>" in line and result <> ''):
                    result = re.sub(r'\s+', ' ', result).strip().split(' ')
                    length = len(result);
                    if(length > 2):
                        for element in result:
                            if '.log' in element:
                                serverName = result[0]
                                path = element
                                applogs.append({"serverName":serverName, "hostip": self.__hostip, "path":path, "ver":1});
                                print "%s - %s" % (serverName, path)
                        pass
                else:
                    match = pattern.match(line)
                    if match:
                        line = re.sub(pattern, ' ', line)
                        line = re.sub(r'\"|\"', '', re.sub(r'%Y%m%d', '*', line))
                        result += line
                    else:
                        pass
        return applogs
        config.close()
           
if __name__ == '__main__':
    if len(sys.argv) != 4:
        print 'Muste be input hostip , containerName and output realpath !!!'
        exit()
    hostip = sys.argv[1]
    containerName = sys.argv[2]
    target = sys.argv[3]
    containerNameAlias = re.sub(r'tomcat', "tmct", containerName)
    containerNameAlias = re.sub(r'timer', "tmr", containerNameAlias)
    containerNameAlias = re.sub(r'jboss', "jbs", containerNameAlias)
    containerNameAlias = re.sub(r'apache', "pch", containerNameAlias)
    if containerName == "apache":
        applogs = Extracter(hostip, containerName).extractApacheConf()
    else:
        applogs = Extracter(hostip, containerName).extractApplicationConf()
    if len(applogs) == 0:
        print '【%s - %s】 Not found logs!!!' % (hostip, containerName)
        exit()
    beatlog = []
    #自定义
    beatlog.append({"module": "middle", "app": "filebeat", "logid": "3000", "logname": "filebeat", "container": "go", "assigner": "evis", "hostip": hostip, "path": "/data/logs/filebeat/filebeat-*/filebeat.log"})
    Builder(beatlog, target, "beat", hash(hostip + "beat")).buildFileBeat()
    Builder(applogs, target, containerNameAlias, hash(hostip + containerNameAlias)).buildFileBeat()
    print "【%sfilebeat-%s.yml】is OK" % (target, containerNameAlias)

```

## 参考资料

* [Filebeat-configuration-details](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-configuration-details.html)