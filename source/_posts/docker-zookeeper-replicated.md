---
title: 利用Docker搭建本地Zookeeper集群
date: 2019-05-06 22:52:10
categories: 
	- Docker
tags:
	- Docker
	- Zookeeper
	- 容器
---

# 背景

最近在学习Dubbo的相关知识，使用了Zookeeper集群作为注册中心，根据Zookeeper官网提供的[Getting Started Guide](http://zookeeper.apache.org/doc/current/zookeeperStarted.html)搭建了一个单机版集群。但此方式首次操作相对较麻烦，需要手动建立三个zoo.cfg文件，同时每次启动和关闭时都需要依次关闭三个Zookeeper服务。

正巧，趁五一假期在家里了解了下Docker相关的基本概念和一些简单的使用，想着正好可以使用Docker来搭建一个本地的Zookeeper集群，这样不仅可以熟练下Docker的基本使用，同时也能更方便地使用ZooKeeper。

# Zookeeper Image

起初设想的是基于Docker的[CentOS Image](https://hub.docker.com/_/centos)编写一个[Dockerfile](https://docs.docker.com/engine/reference/builder/)用来制作[Zookeeper Image](https://hub.docker.com/_/zookeeper)，谁知Docker官方已经提供了Zookeeper的镜像，直接从[Docker Hub](https://hub.docker.com/)下载即可。

直接通过*docker image pull*拉取Zookeeper Image即可，我这里拉取3.4.14版本，你也可以根据自己的需求拉取其他版本：

![获取Zookeeper镜像](/images/2019/docker-zookeeper-pull.png)

# 配置文件

获取到Zookeeper镜像后可以直接使用*docker run*命令来启动一个Zookeeper服务，不过我们这里是要启动一个Zookeeper集群，则考虑使用[docker-compose](https://docs.docker.com/compose/)命令，使用此命令需要编写一个yaml配置文件，此配置文件默认名为*docker-compose.yml*或*docker-compose.yaml*。

因此，我在本地机器上新建一个*docker-compose.yaml*文件。同时为了对Zookeeper集群数据做相关的备份，我建立了一个data目录以免删除Docker 容器之后集群数据就丢失了。最终的文件结构如图所示：

![文件结构](/images/2019/docker-zookeeper-directory.png)

编写的*docker-compose.yaml*配置文件内容如下：

```yml
version: '3.7'

services:
  zoo1:
    image: zookeeper:3.4.14
    restart: always
    hostname: zoo1
    ports:
      - 2181:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - "/Users/wang/zookeeper/data/zoo1:/data"

  zoo2:
    image: zookeeper:3.4.14
    restart: always
    hostname: zoo2
    ports:
      - 2182:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=0.0.0.0:2888:3888 server.3=zoo3:2888:3888
    volumes:
      - "/Users/wang/zookeeper/data/zoo2:/data"

  zoo3:
    image: zookeeper:3.4.14
    restart: always
    hostname: zoo3
    ports:
      - 2183:2181
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=0.0.0.0:2888:3888
    volumes:
      - "/Users/wang/zookeeper/data/zoo3:/data"
```

这里对文件中比较重要的几点做一点解释：

- ports中2181:2181、2182:2181、2183:2181分别表示将本机的2181、2182、2183映射到三台Docker容器的2181端口；
- 默认情况下Zookeeper服务会将数据写入到Docker容器的/data目录下，因此这里对其做了一个映射，分别映射到之前建立的data目录下的zoo1、zoo2和zoo3；
- ZOO_MY_ID和ZOO_SERVERS是Zookeeper镜像定义的几个变量，更多的变量可以查看[Zookeeper Dockerfile](https://github.com/31z4/zookeeper-docker/blob/7742c89bd51c569df89dbcac395c79489c044171/3.4.14/Dockerfile)或者[Zookeeper镜像首页](https://hub.docker.com/_/zookeeper)

# 启动集群

编写完*docker-compose.yaml*文件之后则可以启动Zookeeper集群了，可以在*docker-compose.yaml*文件所在目录下直接执行*docker-compose up*启动集群，也可以通过*docker-compose -f docker-compose.yaml up*指定配置文件启动。

![启动Zookeeper集群](/images/2019/docker-zookeeper-up.png)

执行启动命令后，可以看到控制台输出了很多日志信息，我们可以直接使用*ctr + c*来终止运行，下次再次启动时直接使用*docker-compose start*即可，这样服务就在后台运行了；之后可以使用*docker-compose stop*来停止运行。

![停止和启动集群](/images/2019/docker-zookeeper-start-stop.png)

启动集群后，可以查看本机data目录下多了myid等文件数据，说明正确地将集群数据保存到本机了。

![Zookeeper数据](/images/2019/docker-zookeeper-data.png)

# 集群操作

集群启动后，可以通过*docker-compose ps*来查看集群的状态信息：

![docker-compose ps](/images/2019/docker-zookeeper-ps.png)

当然也可以通过*docker-compose logs*来查看集群的日志信息，有关*docker-compose*命令的更多使用这里就不一一介绍了，感兴趣的读者可以阅读[Docker Compose官方文档](https://docs.docker.com/compose/)。

最后，我们可能需要登录到Zookeeper集群做一些维护性的工作，之前提到过集群启动时会绑定到当前机器的2181、2182和2183端口，因此，我们只需要在本机使用zkCli.sh登录到集群即可。

![登录到Zookeeper集群](/images/2019/docker-zookeeper-client.png)

