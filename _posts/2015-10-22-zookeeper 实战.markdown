---
title: zookeeper 实战
notebook: 日常
tags:笔记
---
Zookeeper环境搭建
=================
下载

    $ wget http://apache.forsale.plus/zookeeper/zookeeper-3.4.9/zookeeper-3.4.9.tar.gz  

配置`conf/zoo.cfg`

    # 客户端连接zookeeper服务的端口。这是一个TCP port。
    clientPort=2181
    # 内存数据结构的snapshot，便于快速恢复
    dataDir=/data
    # 存放顺序日志(WAL)
    dataLogDir=/datalog
    # 集群配置
    server.1=192.168.8.154:20881:30881
    server.2=192.168.8.153:20881:30881
    server.3=192.168.8.152:20881:30881

> dataLogDir如果没提供的话使用的则是dataDir,为了达到性能最大化，一般建议把dataDir和dataLogDir分到不同的磁盘上，这样就可以充分利用磁盘顺序写的特性。上面的配置中有两个TCP port。 后面一个是用于Zookeeper选举用的， 而前一个是Leader和Follower或Observer交换数据使用的。

启动与连接zookeeper

    # 启动服务器
    $ ./bin/zkServer.sh start
    ZooKeeper JMX enabled by default
    Using config: /home/dmall/app/zookeeper-3.4.9/bin/../conf/zoo.cfg
    Starting zookeeper ... STARTED
    # 启动客户端
    $ ./bin/zkCli.sh -server 192.168.8.154:2181
    Connecting to 192.168.8.154:2181
    # 查看服务器状态
    $./ bin/zkServer.sh status


报错`Invalid config, exiting abnormally`
==================
集群启动时，需要在dataDir目录下创建myid文件（touch myid），并按照集群顺序，依次在三台主机上的该文件下填写1,2,3(echo "1" > myid)

zookeeper启服务的时候最好是依从myid从小到大依次重启
===================
今天启动服务器后，发现客户端连接不上，错误如下

     Unable to read additional data from server sessionid 0x0, likely server has closed socket, closing socket connection and attempting reconnect

查看服务器日志

    $ tail -200f zookeeper.out 
    Have smaller server identifier, so dropping the connection: (3, 1)

意思是有更小的服务id，dropping了连接

> google结论：重启服务的时候最好是依从myid从小到大依次重启, 因为这个里面又涉及到zookeeper另外一个设计.zookeeper是需要集群中所有集群两两建立连接, 其中配置中的3555端口是用来进行选举时机器直接建立通讯的端口, 为了避免重复创建tcp连接,如果对方myid比自己大，则关闭连接，这样导致的结果就是大id的server才会去连接小id的server，避免连接浪费.如果是最后重启myid最小的实例,该实例将不能加入到集群中,因为不能和其他集群建立连接, 这时你使用nc命令, 会有如下的提示: This ZooKeeper instance is not currently serving requests. 在zookeeper的启动日志里面你会发现这样的日志: Have smaller server identifier, so dropping the connection. 如果真的出现了这个问题, 也没关系, 但是需要先将报出该问题的实例起着,然后按照myid从小到大依次重启zk实例即可. 是的,我们确实碰到了这个问题, 因为我们稍后会将机房3的那个zk实例的myid变为0,并最后加入到11台实例的集群中,最后一直报这个问题. [参见](http://siye1982.github.io/2015/06/16/zookeeper/)