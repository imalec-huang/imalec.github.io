screen窗口管理器常用操作

screen是一个可以在多个进程之间多路复用一个物理终端的窗口管理器。

 

1.  创建新的screen会话

screen [command] [-S name]

2.  Detach 会话

screen –d [screen name]

3.  Reattach 会话

screen –r screen-name

4.  查看所有的screen会话

screen –ls

 

进入screen会话后，可在会话中创建多个窗口（window），并对窗口进行管理，管理命令以ctrl + a开头。

 

ctrl + a + c：创建新窗口（create）

ctrl + a + n：切换至下一个窗口（next）

ctrl + a + p：切换至上一个窗口（previous）

ctrl + a + w: 列出所有窗口

ctrl + a + A: 窗口重命名

ctrl + a + d：detach当前会话

ctrl + a + [1-9]: 切换到指定窗口（1-9为窗口号）

ctrl + d：退出（关闭）当前窗口

 

进入screen后，当按下tab键时，会闪屏，可通过ctrl + a & ctrl + g来停止当前screen的闪屏，如果要对所有的screen生效，在~/.screenrc中加入vbell off。



改变screen配置：

caption always "%{= kw}%-w%{= BW}%n %t%{-}%+w %-= @%H - %Y-%m-%d %c"

vim /tmp/screenrc

screen -c /tmp/screnrc -S zl