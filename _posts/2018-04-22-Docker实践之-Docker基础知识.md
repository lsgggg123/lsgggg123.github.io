---
layout: post
title: 'Docker实践之一 -- Docker基础知识'
subtitle: '如果你是一个服务端开发人员，那么这几年你一定听说过Docker的火爆，那么火爆的Docker到底是什么？Docker能做些什么，又是如何方便开发运维过程的？这篇文章将通过我自己对Docker的理解尝试着回答上述几个问题。'
date: 2018-04-22
categories: 技术
tags: 技术 运维 docker linux 
cover: '/assets/img/20180422/docker-facebook.png'
---


如果你是一个服务端开发人员，那么这几年你一定听说过Docker的火爆，那么火爆的Docker到底是什么？Docker能做些什么，又是如何方便开发运维过程的？这篇文章将通过我自己对Docker的理解尝试着回答上述几个问题。

### Docker是什么？
我们看下Docker官方给出的对Docker是什么的回答：

> Docker is an open platform for developers and sysadmins to build, ship, and run distributed applications, whether on laptops, data center VMs, or the cloud.

这话就不翻译了，翻译出来就没意思了。

为了简单理解，你可以认为Docker是一种虚拟机技术。一提虚拟机大家都很明白，虚拟机通过软件在底层虚拟出来完整的一套硬件，内存，CPU，硬盘什么的，然后你可以把虚拟出来的硬件当做真实的硬件，虚拟出来的系统和真实的系统没有任何区别，你新建了一个虚拟机，那么你就多拥有了一台机器。虚拟机通过完整的虚拟一套硬件，来实现环境的隔离，也就是说，你在虚拟机里面做的一切事情都和你原来的那个操作系统无关，你可以随意删除系统文件、裸奔、运行来路不明的病毒木马，受到影响的只是你虚拟出来的那台机器，你的环境是隔离的。对于虚拟机，请牢记这两个关键词：硬件虚拟 + 环境隔离。

虚拟一套硬件的成本是很高的，所以虚拟机很吃硬件资源。Docker走了另外的一条路，没有再虚拟一套完整的硬件，反正只要我把环境给隔离好了，硬件就直接用原生的就行了。所以，对Docker来说，关键字只有环境隔离，没有硬件虚拟。事实上，Docker的环境隔离做的如此之好，以至于你产生了你是在虚拟机里面这样一种幻觉。Docker的隔离是利用Linux的LXC（Linux Container）等技术为基础的，在LXC的基础上Docker进行了进一步的封装，让用户不需要去关心LXC的管理，使得操作更为简便。明白了这一点，就可以解释Docker的两个特点：1.Docker启动一个容器（可以把容器理解为一个虚拟的操作系统）极快，只需要数秒的时间，而且除了容器本身运行的应用之外，容器本身几乎不占用资源。你甚至可以直接把容器当成应用使用，这没毛病。因为Docker本身直接复用本地主机的操作系统，而传统的虚拟机还要额外运行一个完整的OS。2.Docker目前（2016-10-31）只能运行在Linux上面，尽管Docker一直在说将来有计划原生支持其他的操作系统，也推出了Mac版和Windows版的Docker应用，实际上由于底层基础技术是LXC，所以这些非Linux版本的实际上都是虚拟了一个Linux环境。

Docker是什么应该有一个大概的了解了，但是为了后续的深入，还是要介绍下三个新的名词，这三个名词是Docker的三个核心概念：

#### 镜像
Docker 镜像就是一个只读的模板。

例如：一个镜像可以包含一个完整的ubuntu操作系统环境，这样的镜像称之ubuntu镜像。镜像里面也可以在操作系统的基础上安装一个或多个应用程序，例如ubuntu + mysql，这样的镜像我们一般称之为mysql镜像。

镜像可以用来创建 Docker 容器。

Docker 提供了一个很简单的机制来创建镜像或者更新现有的镜像，用户甚至可以直接从其他人那里下载一个已经做好的镜像来直接使用。

#### 容器

Docker利用容器来运行和隔离应用，类似一个轻量级的沙箱。

容器是从镜像创建的运行实例。它可以被启动、开始、停止、删除。每个容器都是相互隔离的、保证安全的平台。

可以把容器看做是一个简易版的 Linux 环境（包括root用户权限、进程空间、用户空间和网络空间等）和运行在其中的应用程序。

镜像是只读的，容器在启动的时候创建一层可写层作为最上层，镜像本身保持不变。

#### 仓库

简单理解成获取镜像的地方，打个比方，镜像是jar包，仓库就是maven仓库，maven仓库有公开的，也有私人的，Docker也是一样，最大的公开仓库是Docker Hub，同时也是官方的Docker仓库，不过国内的访问速度实在令人发指，阿里云好像和Docker Hub合作了，但是目前的镜像拉取速度还是照旧，可能需要时间，这里推荐下网易的蜂巢(https://c.163.com/hub#/m/library)，镜像拉取速度很快。有了仓库，Docker镜像的分发就十分容易了。

好了，理解了上面的几个概念，我们可以继续后面的主题了。

### Docker能做些什么，又是如何方便开发运维过程的
前面我们已经知道了Docker的很多特点，可以用来做环境隔离，而且启动运行停止的资源消耗极低，Docker内运行应用程序接近原生运行（Docker内启动一个nginx只占5M不到的内存）。现在回想一下一个新的应用上线的情况，你应该先发邮件给运维的人申请机器，申请多少台是个问题，为了保证业务的稳定运行，你必须按照业务的峰值进行申请，如果一个业务的峰值和低谷相差十分大，那么你可能申请的很多机器大部分时间都浪费在那里。这还不算完，你想一下，你的机器申请下来之后，你要安装必要的应用，搭建环境，每一台都是如此！这对运维来说简直是噩梦。现在好了，你有了Docker，你把你的业务需要的环境打包成一个镜像，业务高峰期来的时候多启动几个Docker容器，高峰期过了停止几个容器释放出来内存可以给别的业务用，而且这一过程完全可以自动化，公司大了，这样省下的机器费用简直就是一笔巨款啊。除了这个，你应该还能想到Docker能做的事情就是Docker可以快速搭一个集群环境出来，例如我要测试一个Redis集群，没有Docker的时候我可能要在一台服务器上安装多个Redis，依次进行配置，然后依次启动它们，折腾半天才能搞好，而现在有了Docker，我直接拉取一个Redis的镜像，直接启动这个镜像，就有了一个Redis服务器了，我再次启动这个镜像，又有了一个Redis服务器，这一切都非常方便。还有呢，通过Docker仓库，我可以很方便的把我的应用需要的环境打包起来共享给别人，这点你可以回想下你入职新公司从搭建一个开发环境到所有环境可以总用使用了多少时间？你的应用要上线，各种环境和配置设置好要多少时间多少次数？有了Docker，你只需要配置一次，把镜像推到Docker仓库里，别人就可以直接使用你的环境啦。

### Docker基础命令

前面说了那么多，你一定心动了，等不及要开始使用Docker做点什么了，别急，现在我们先进入Docker的基础命令部分。

#### Docker安装

Docker的安装需要条件的，你的内核要大于等于3.10，使用命令uname -r命令可以查看机器的内核，前两位要是小于3.10，您自己洗洗睡或者去想法儿升级内核随您。另外说一句，要是CentOS的话，最好使用CentOS7的，再不济也得是6.5的。

推荐Docker的一键安装方法：

`sudo curl -sL https://get.docker.io/ | sh`

还有阿里云提供的一个脚本，和上面的没有什么区别：

`sudo curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh`

要是这个一键安装失败了，看下面的方法。

CentOS6.5:

`$ sudo yum install http://mirrors.yun-idc.com/epel/6/i386/epel-release-6-8.noarch.rpm`

`$ sudo yum install docker-io`

CentOS7：

```XML
$ sudo yum install docker
```

Ubuntu 14.04及后续版本：

```XML
$ sudo apt-get update

$ sudo apt-get install -y docker.io

$ sudo ln -sf /usr/bin/docker.io /usr/local/bin/docker

$ sudo sed -i '$acomplete -F _docker docker' /etc/bash_completion.d/docker.io
```

或者

```XML
$ sudo apt-get install apt-transport-https

$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9

$ sudo bash -c "echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"

$ sudo apt-get update

$ sudo apt-get install lxc-docker
```

Ubuntu 14.04 之前版本

升级内核：

```XML
$ sudo apt-get update

$ sudo apt-get install linux-image-generic-lts-raring linux-headers-generic-lts-raring

$ sudo reboot
```

然后重复上面的步骤即可。

注意！安装完成了，要启动Docker服务，不然你后面的命令都报错，我看的GitBook上面yeasy的那个，google搜一下排第一的，他就没说安装好了要启动Docker服务，坑爹。

启动服务命令：

`sudo service docker start`

输入下面命令，如果显示Docker版本信息，则安装成功：

`docker version`

直接输入docker可以看到所有的docker命令和简要说明。



#### 镜像相关操作

> docker images 列出所有的镜像

> docker search 镜像名称 搜索镜像，目前没有办法连同标签一起搜索，我一般都是去官网上看有哪些标签可选

> docker pull 镜像名[:镜像tag] 从Docker Hub拉取一个镜像，镜像tag可选，不写就是latest，如果不想从Docker Hub拉取，镜像名写全部的url即可，例如：docker pull hub.c.163.com/public/ubuntu:16.04

> docker tag 镜像名[:镜像tag] 镜像名[:镜像tag] 重新给镜像分配镜像名和tag，和上面的可以配合使用，如果我从网易蜂巢拉了一个ubuntu:16.04，那么它的镜像名字是hub.c.163.com/public/ubuntu，每次操作我都要完整的输入这个名字，那我就可以用docker tag给它重新定义一个短的名字

> docker rmi 镜像Id 删除一个镜像，删除一个镜像需要这个镜像的所有容器都删除，删除镜像的容器又需要所有的容器都停止。

> docker commit 镜像Id [仓库[:镜像tag]] 对容器的修改打包成镜像，容器如果不commit，无法数据持久化

> docker build Dockerfile 通过Dockerfile文件创建镜像，Dockerfile语法后续提供

> docker inspect 查看镜像的详细信息

> docker push 上传镜像到仓库，注意这个命令和docker tag一起使用，docker tag直接把



#### 容器相关操作

> docker create -it 镜像名[:镜像tag] 创建一个容器，创建的容器处于停止状态

> docker ps 查看处于运行中的镜像，使用-a命令可以看到所有容器，这个命令可以看到容器id

> docker run -t -i 镜像名称 --name xxx /bin/bash 新建并启动容器，这是最重要的一个命令，这里重点分析下这个命令

```XML
-i 让容器的标准输入保持打开

-t 分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上

-d 后台的方式启动一个容器，使用了-d，就别加-it参数了

--name xxx 给容器起一个别名，相同的别名不管运行中还是已经停止了只能起一个，加了别名后，操作容器id就可以用别名代替

/bin/bash 启动容器的bash终端，也可以执行shell脚本等等其他操作，这里说一句，很多操作系统镜像可以不加这一句，默认就启动一个bash终端，然而绝大部分的应用镜像（例如nginx, mysql, redis镜像）不加这一句默认启动的是镜像里安装的应用，加上了/bin/bash就不启动对应的应用了，可以在容器的/bin/bash里面再启动容器或者修改配置（修改了配置别忘了commit保存为镜像）

-v, -p, -link后面再说
```

如果使用-it参数，则进入容器后这个时候就无法使用docker命令了，已经进入了容器内部。输入exit，或者按下ctrl + d 停止并退出容器，如果想退出但不停止容器，这个时候可以按下ctrl + p, ctrl + q回到宿主机。回到宿主机，如果想再次进入容器，可以用docker attach 容器id，就可以再次进入容器。

docker stop 容器id 停止一个容器，这里说一句，容器id是一串数字+字母，很多命令需要跟上容器id，容器id不需要全部都写上，只要不引起歧义，可以只写前几位

> docker start 容器id 开始一个处于停止状态的容器

> docker restart 容器id

> docker rm 容器id 删除一个容器，要求容器处于停止状态，这里多说一句，容器的创建、停止、删除都是很轻量级的，不会占据很多时间和资源，所以容器不用了就停止删除，用的时候就创建开始。

> docker logs 容器id 查看容器的日志输出，适用于-d启动和后台运行状态的容器

> docker top 容器id 查看容器的资源占用情况

一些实例：

`docker run -it --name centos -v /Users/lsgggg123/docker:/root/docker centos`

`docker run --name mysql -e MYSQL_ROOT_PASSWORD=123456 -d -p 3306:3306 mysql`

`docker run -it nginx --name nginx /bin/bash`

`docker run -d -p 8090:80 nginx --name nginx`


#### 仓库相关操作

> docker push 上传镜像到仓库，注意这个命令和docker tag一起使用，docker tag直接把要上传的仓库路径作为镜像名，才能push

> docker login -u xxx -p url

> docker search，docker pull 这两个上面已经说了


#### 数据持久化相关操作

> docker run -v path1 创建一个数据卷并挂载到容器里，路径是path1，在一次 run 中多次使用可以挂载多个数据卷。数据卷只对一个容器有效，可以保证容器停止再启动后数据持久化，删除容器后数据卷不可用。

> docker rm -v 容器id 删除容器，并删除容器的数据卷

> docker run -v path1:path2 把宿主机的path1挂载到容器的path2，容器访问path2相当于访问path1。注意这个功能虽然支持文件，但是最好只对路径使用，不要对文件使用。


#### 容器间共享数据：

> docker run -d -v /data --name data centos

> docker run -d --volumes-from data --name data1 centos

> docker run -d --volumes-from data --name data2 centos

使用 --volumes-from 参数所挂载数据卷的容器自己并不需要保持在运行状态。



#### 网络相关操作

> docker run -p port1:port2 把宿主机的端口port1映射到容器的端口port2上，可以加多个-p映射多个端口

容器互联：

> docker run -d --name db mysql 先启动一个容器，这个容器假设为提供数据库服务的容器，那么很自然的，其他的容器想要访问这个容器的数据库

> docker run -d -P --name web --link db:db tomcat 启动一个tomcat的web服务器，可以直接连接数据库容器，使用db可以代替数据库容器的ip地址直接访问

Docker实战部分暂时先写这些，下篇文章再写一下如何使用Docker部署一个Spring Boot应用


更多的Docker的基础知识请看这边书：[《Docker —— 从入门到实践》](https://www.gitbook.com/book/yeasy/docker_practice/details)

### 参考

[http://bg.biedalian.com/2014/11/20/docker-start.html](http://bg.biedalian.com/2014/11/20/docker-start.html)

[http://wiki.jikexueyuan.com/project/docker-technology-and-combat/what.html](http://wiki.jikexueyuan.com/project/docker-technology-and-combat/what.html)

[http://dockone.io/article/133](http://dockone.io/article/133)