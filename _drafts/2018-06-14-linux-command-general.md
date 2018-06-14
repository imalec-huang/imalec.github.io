---
layout: post
title:  "常用的一些linux命令"
date:   2018-06-14 19:19:00
categories: DevOps
tags: linux shell
---

* content
{:toc}

常用linux命令记录。





## screen
screen -S name
screen -ls
screen -r name
## sed
批量替换文件内容
ls -al|awk '{print $9}'|grep conf|xargs -i sed -i 's/1datatech.com/1datatech.cn/g' {}

批量修改文件名
rename 's/_com/_cn/' *.conf
inux下tar命令解压到指定的目录 ：
#tar zxvf /bbs.tar.zip -C /zzz/bbs   
//把根目录下的bbs.tar.zip解压到/zzz/bbs下，前提要保证存在/zzz/bbs这个目录 
这个和cp命令有点不同，cp命令如果不存在这个目录就会自动创建这个目录！
附：用tar命令打包
例：将当前目录下的zzz文件打包到根目录下并命名为zzz.tar.gz
#tar zcvf /zzz.tar.gz ./zzz
踢用户下线 
w 查看用户tty
pkill -kill -t usertty
新增用户创建用户目录、加入sudo组、指定登陆shell -m -s -d -G
useradd -m -s /bin/bash -d /home/web -G sudo web
passwd web
修改用户名 -l -d -m 修改用户ID 修改组ID
 usermod -l 新用户名  -d /home/新用户名 -m 老用户
usermod -u 1000 web 
groupmod -g 1000 web
删除用户及目录-f 强制，即使登陆状态
userdel -r web
永久修改hostname
 vi /etc/hostname 
vi /etc/hosts
reboot
指定用户运行命令 
su - root -s /bin/sh -c "/usr/local/nginx/sbin/nginx"
sudo -u weblogic /home/weblogic/sbin/starup.sh
硬盘挂载
#fdisk -l
#fdisk xvde1
fdisk可以用m命令来看fdisk命令的内部命令；
a：命令指定启动分区；
d：命令删除一个存在的分区；
l：命令显示分区ID号的列表；
m：查看fdisk命令帮助；
n：命令创建一个新分区；
p：命令显示分区列表；
t：命令修改分区的类型ID号；
w：命令是将对分区表的修改存盘让它发生作用。
n -> p ->enter*3 >w 
Command (m for help):n
Command action
　　   e    extended                  //输入e为创建扩展分区
　　   p    primary partition (1-4)      //输入p为创建逻辑分区
p
Partion number(1-4)：1      //在这里输入l，就进入划分逻辑分区阶段了；
First cylinder (51-125, default 51):   //注：这个就是分区的Start 值；这里最好直接按回车，如果您输入了一个非默认的数字，会造成空间浪费；
Using default value 51
Last cylinder or +size or +sizeM or +sizeK (51-125, default 125): +200M 注：这个是定义分区大小的，+200M 就是大小为200M ；当然您也可以根据p提示的单位cylinder的大小来算，然后来指定 End的数值。回头看看是怎么算的；还是用+200M这个办法来添加，这样能直观一点。如果您想添加一个10G左右大小的分区，请输入 +10000M ；
Command (m for help): w                     //最后输入w回车保存。
#mkfs.ext4 /dev/xvde
#mkdir /data
#mount /dev/xvde1 /data
#vim /etc/fstab
/dev/xvde1(磁盘分区)  /data（挂载目录） ext3（文件格式）defaults  0  0
sed -i '$a \/dev/xvde1 /data ext4 defaults 0 0' /etc/fstab