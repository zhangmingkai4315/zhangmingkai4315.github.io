---
layout: post
title: Docker使用记录
date: 2018-11-17T11:32:41+00:00
author: mingkai
permalink: /2018/11/docker-usage
categories:
  - docker
---

Docker将用户软件放在容器中，使应用程序的复杂度和基础设施隔离，基础设施本身变得更加简单，应用程序也更易于部署。相对于虚拟机，执行效率和速度更快，内存是共享的而不是预先分配的。用户的应用程序可以更低的成本运行，也可以按照需要的方式进行构建架构设计，而不再受原有基础设施的制约。

Docker出现之前，开发一般基于虚拟机或者Vagrant工具进行，而测试人员则通过Jenkins工具进行测试，部署上线的则是运维人员使用Chef或者Ansible等工具，使用和管理成本较高。现在则可以都通过Docker构建的方式来完成整个开发，测试和部署。

>  Docker指的是早起在码头进行货物装配的工人，如果直接放置大小各异的东西到货船上，势必造成混乱和难以管理，但是集装箱的引入似的装配和卸载等都比较容易

Docker的优势主要由

- 替代虚拟机，使用命令行可以更容易的脚本化系统
- 软件原型，快速的启动一个软件
- 打包软件，镜像是没有依赖的，直接打包和构建更加方便
- 微服务设计
- 网络建模，构建一个复杂的网络环境
- 可放置于笔记本上的全栈系统部署
- 降低团队沟通成本
- 文档化软件依赖和入口程序
- 持续交付管理

![](http://cdn.rancher.com/wp-content/uploads/2017/02/16175231/VMs-and-Containers.jpg)



### 1. Docker基础



#### 1.1 容器和镜像

容器运行镜像定义的系统，**镜像则由一个或者多个层（差异集合）加上一些Docker元数据组成**。这些元数据包括：环境变量，端口映射以及卷等信息。文件则包含工具的副本，语言环境以及库等内容。　当镜像生成容器后，所有文件的变更将利用**写时复制**方式存储在容器中，基础镜像不会收到影响。这些容器继承镜像的文件系统，并利用元数据来确定启动配置。

可以类比为程序和进程，进程是一个被执行的程序。或者类比为面向对象编程中类和对象的概念。



#### 1.2 安装DockerCE

在ubuntu16.04版本以上安装Docker, 内核支持OverlayFS，Docker将使用overlay2存储驱动。确保linux的内核版本大于4.0，这种overlay2驱动方式相比原原来的aufs或者devicemapper效率更高，同时为了提升效率可以使用更快的存储介质比如ssd,以及使用卷来作为存储方式(Volume)

安装可参考官方的文档[Get Docker For Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/)，**注意最后添加用户权限，提供无需sudo即可执行命令**

```
sudo usermod -aG docker YOUR-USERNAME
# sudo addgroup -a YOUR-USERNAME docker
```

#### 1.3 定义Dockerfile

下面是一个Dockerfile的配置实例，我们使用docker来运行一个node应用, 定义的Dockerfile文件如下：

```dockerfile
FROM node
MAINTAINER zhangmingkai.1989@gmail.com
RUN git clone -q https://github.com/docker-in-practice/todo.git
WORKDIR todo
RUN npm install > /dev/null
RUN chmod -R 777 /todo
EXPOSE 8000
CMD ["npm","start"]
```

在执行```docker build . ```后会创建对应的镜像文件，构建镜像过程中会创建一些中间的容器，这些容器基本上在使用结束后都会被直接销毁，仅仅留下对应的镜像文件，镜像文件会被用于下次构架的缓存，　最终输出的镜像具有一个ID值，但是我们可以使用tag的方式对镜像进行标记。

```shell
Removing intermediate container ba8e3519b3d5
 ---> 580217127938
Successfully built 580217127938
➜  CH-1 docker tag 580217127938 todoapp
➜  CH-1 docker images        
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
todoapp             latest              580217127938        3 minutes ago       979MB
```

一些常用的docker命令:

- docker run -p 8000:8000 --name todo-example todoapp 启动容器管理，-p 宿主机端口:容器端口
- docker ps　<-a>                      显示容器的状态
- docker diff <容器>　               显示自实例化后的文件变化
- docker rm -f container            删除运行过的容器
- docker run --name wordpress --link wp-mysql:mysql -p 10010:80 -d wordpress 　链接一个mysql容器到当前的容器，实现容器间的通信，不用暴露给宿主机。
- docker build -t 镜像名称　.  　从当前目录读取dockerfile生成镜像
- docker commit 容器名　　　  使用容器保存状态到一个新的镜像，提供后期使用（比如容器内做了一些新的修改）
- docker tag 镜像ID 镜像名称 

#### 1.4 Docker分层设计

docker的分层设计，可以使得应用占用的资源更少，docker在内部使用写时复制技术，允许占用的空间更少，每个启动的容器仅仅包含必要的增量数据，而不是整个的应用镜像的数据，**不需要复制任何的东西，所有的数据已经存储在镜像中。**

![](https://docs.docker.com/storage/storagedriver/images/container-layers.jpg)

> **写时复制**：在模板创建数据的时候，仅仅在数据发生变化的时候才复制出来，而不是复制整个的数据集合。

一个新的镜像，往往由其他多个镜像组成，这些镜像之间是共享的，因此占用的资源也很少。



### 2.  Docker架构设计

安装在主机上的docker其实包含两个部分，docker守护进程和docker客户端，两者之间通过api接口的方式进行通信，同时外部还有类似于docker hub的注册中心，用于管理镜像，提供镜像的下载等功能。docker守护进程负责和docker客户端的请求，并和其他的注册中心进行通信，同时负责管理容器和镜像



![](https://i0.wp.com/knoldus.blog/wp-content/uploads/2018/04/high-level-overview-of-docker-architecture.png)

docker守护进程默认情况下只能接收本地的docker客户端的请求，限制通过/var/run/docker.sock的套接字进行访问，但是可以使用```docker daemon -H tcp://0.0.0.:2375```使其运行在开放的网络中，提供所有的网络内的客户端请求，不过最好限制使用的IP地址，防止一些安全问题。

```
docker run -d -i -p 1234:1234 --name daemon ubuntu nc -l 1234
```

上面的这种方式中-d代表后台运行　-i 代表允许telnet交互式方式访问, 同时可以赋予容器一定的重启策略

- --restart=always　　　出错后一直重试
- --restart=no　　　　　不会运行重试
- --restart=on-failure:10 只有在超过重试次数后才退出



#### 2.1 使用socat诊断API调用

下面是使用socat来获取api数据的操作，创建一个临时的sock文件，并转发到docker默认的sock套接字上，任何发送到临时套接字的数据都会显示出来并被转发出去。输出包含整个的查询发送和接收的API数据。

```shell
sudo apt-get install socat
docker ps -a 
sudo socat -v UNIX-LISTEN:/tmp/dockerapi.sock UNIX-CONNECT:/var/run/docker.sock &
# docker -H unix:///tmp/dockerapi.sock ps -a
sudo docker -H unix:///tmp/dockerapi.sock ps -a

```

#### 2.2 建立本地的docker注册中心

直接选择使用容器的方式启动一个注册中心，绑定本地的5000端口，设置本地的文件目录映射，同时配置重启策略。

```
$ docker run -d -p 5000:5000 -v $HOME/registry:/var/lib/registry --restart=always --name registry registry:2
```

具体的注册中心代码可参考[distribution](https://github.com/docker/distribution)

另外可以配置注册中心的数据到一个外部的存储环境比如swift, s3或者gcs等对象存储

任何想要连接私有注册中心的，可以配置**--insecure-registry HOSTNAME**到docker守护启动项



### 3. 开发环境使用Docker

docker并不是一种虚拟机技术，没有虚拟任何的硬件或者操作系统，没有被约束于指定的硬件限制，仅仅是吧服务运行的环境给虚拟化。

- docker面向应用，虚拟机面向的操作系统
- docker容器和其他容器共享一个操作系统，虚拟机则管理它自己的操作系统
- docker容器被设计成运行一个主要进程，而不是管理多组进程

如果之前依赖于虚拟机运行，可以通过虚拟机转镜像生成容器的方式迁移到docker运行环境中，具体的操作方式可以使用qeuｍ-nbd工具来完成，将生成的tar包使用docker import的命令转换为镜像。另外可以通过ADD的方式引入（借助以scratch镜像）。

#### 3.1 容器中运行多个进程

docker不仅仅能够运行单一的进程，用户在使用的时候可以将多个进程同时打包到一个容器中，

##### 3.1.1  使用runit管理多进程

可以参考[phusion/baseimage](https://github.com/phusion/baseimage-docker)来管理多个进程，用户可以设置基于该镜像写dockerfile,并且使用一个统一的管理工具来管理多个进程的运行, 比如下面我们运行一个后台程序，并将它加入到启动程序中，这里使用的是runit来管理进程。

```shell
#!/bin/sh
# `/sbin/setuser memcache` runs the given command as the user `memcache`.
# If you omit that part, the command will be run as root.
exec /sbin/setuser memcache /usr/bin/memcached >>/var/log/memcached.log 2>&1
```

对应的dockerfile文件加入

```dockerfile
FROM phusion/baseimage

RUN mkdir /etc/service/memcached
COPY memcached.sh /etc/service/memcached/run
RUN chmod +x /etc/service/memcached/run

CMD ["/sbin/my_init"]
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
```



##### 3.1.2  使用supervior管理多进程

另外的一种方式来管理多进程的是启动superviord服务来管理多个进程，设置superviord服务以前台运行的方式作为容器的主进程。相关的配置如下,启动一个nginx服务器和一个sshd服务器，首先dockerfile如下：

```dockerfile
FROM ubuntu:14.04
MAINTAINER sharad &lt;sharad.aggarwal@tothenew.com&gt;
RUN apt-get update &amp;&amp; apt-get install -y openssh-server apache2 supervisor nginx
RUN mkdir -p /var/lock/apache2 /var/run/apache2 /var/run/sshd /var/log/supervisor
RUN echo 'root:password' | chpasswd
RUN sed -i 's/PermitRootLogin without-password/PermitRootLogin yes/' /etc/ssh/sshd_config
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
EXPOSE 22 80 443
CMD ["/usr/bin/supervisord"]
```

superviord的配置文件如下：

```shell
[supervisord]
nodaemon=true
 
[program:sshd]
command=/usr/sbin/sshd -D
 
[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
priority=900
stdout_logfile= /dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
username=www-data
autorestart=true
```

#### 3.2 dockerfile中的一些问题

在一个dockerfile中，经常使用的更新和安装版本的写法

```
    RUN apt-get update && apt-get install -y \
        package-bar \
        package-baz \
        package-foo
```

如果将起拆开写，则可能导致之后的任何修改重新build的时候，apt-get update一直使用过期的缓存而不去更新。

将同时运行了多个服务的容器dockerfile拆成多个，需要考虑dockerfile本身的缓存是否可以减少不必要的执行。比如下面的

```
COPY app /opt/app
RUN cd /opt/app && npm install  
```

这里由于npm install 执行的操作来自于package.json文件，因此我们可以提取该文件，单独执行npm install

```
COPY app/package.json /opt/app/package.json
RUN cd /opt/app && npm install
COPY app /apt/app
```

参考链接：https://stackoverflow.com/questions/35774714/how-to-cache-the-run-npm-install-instruction-when-docker-build-a-dockerfile

#### 3.3 保存状态

如果我们当前正在利用docker进行辅助开发，所有修改的代码都会被最终编译到镜像中进行保存，用于后期的测试和发布，对于代码的版本修改我们都需要生成一个新的镜像。

使用docker build 命令可以重新打包应用到镜像中，重新发布。但是如果我们进到交互式的容器中执行了很多其他的命令比如加入了新的包，或者安装了新的工具。我们可能需要保存当前的运行环境，则使用docker commit 容器名称即可，该操作将生成一个新的镜像，用于后期的使用。

使用docker tag可以将前面生成的容器进行打标签，赋予一个更有用的名字

```shell
docker run -d -p 8080:80 --name my-static-web nginx-static
docker exec -it af /bin/sh　　　　# 进入容器执行apk add vim 
docker commit af　　　　　　　　　　# 将容器保存成镜像
docker tag 74936e55404f nginx-with-vim　　# 赋予容器一个标签
docker rm -f my-static-web　
docker run -d -p 8080:80 --name my-static-web nginx-with-vim　　# 重新启动容器
```



#### 3.4 给镜像打标签

上面我们给commit的镜像打了新的标签，默认情况下我们从注册中心将拉取latest标签的镜像，但是这个名称不代表最新的或者最近的一次修改，仅仅只是一个标签值。可以给镜像打多个标签，比如latest，v1.0 , v1.0.1等等，这些标签代表的可以是一个相同的镜像

```
➜  CH-3 docker tag nginx-with-vim nginx-with-vim:1.0 
➜  CH-3 docker tag nginx-with-vim nginx-with-vim:1.0.1
➜  CH-3 docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx-with-vim      1.0                 74936e55404f        11 minutes ago      47.4MB
nginx-with-vim      1.0.1               74936e55404f        11 minutes ago      47.4MB
nginx-with-vim      latest              74936e55404f        11 minutes ago      47.4MB
```

同时如果我们有一个内部的镜像注册中心比如：http://docker-hub.example.com 则在push镜像到本地的注册中心的时候可以按照如下的方式：

```shell
docker pull ubuntu
docker tag ubuntu:latest docker-hub.example.com/mike/ubuntu:latest
docker push docker-hub.example.com/mike/ubuntu:latest
```

这样我们就可以直接内部使用ubuntu镜像了，注意打标签的时候必须加入注册中心的域名地址，否则默认会是公共的docker-hub.com



### 4. Docker的管理

#### 4.1 使用bttorrent协议共享数据

如果通过卷的方式将宿主机器的文件映射到容器中，需要确保宿主机器上SELinux策略的问题，如果是enforced则无法直接进行修改，需要调整策略

```
docker run -d -v /var/db/data:/var/data -it debian bash
```

上面的命令将创建一个卷的映射。

如果需要同步数据在多个主机上可以考虑使用bittorrent协议完成，具体的实施如下图：

![](https://jaxenter.com/wp-content/uploads/2015/08/Screen-Shot-2015-08-17-at-10.55.14.png)

#### 4.2 使用数据容器

创建一个数据容器，并不一定需要其运行，仅仅指定容器启动一次即可

```
➜  ~ docker run -v /shared --name data-container busybox
➜  ~ docker run -it --volumes-from data-container ubuntu /bin/bash
root@07976aab0403:/# cd /shared/
root@07976aab0403:/shared# touch hello
root@07976aab0403:/shared# exit
exit
➜  ~ docker run -it --volumes-from data-container ubuntu /bin/bash
root@da779faea606:/# cd /shared/
root@da779faea606:/shared# ls
hello
```

使用数据容器的时候，我们指定一个名称data-container后，该容器就直接退出了，但是容器本身的文件系统已经创建并被保存下来，我们在创建其他容器的时候直接使用**--volumes-from**挂载原来的容器到之前声明的路径上。

多个容器同时使用数据容器的时候要严格的划分好使用的文件路径，防止出现写入的错乱问题。



#### 4.3 使用SSHFS来挂载远程磁盘

使用sshfs可以利用ssh协议来挂载远程服务器上的文件系统，前提是通过root权限操作，且必须安装了FUSE文件内核模块，才能使用。远程服务器仅仅是一个SSH服务器即可，本地的宿主机通过FUSE内核模块映射到本地文件中。
![](https://www.cyberciti.biz/media/new/images/faq/2015/03/sshfs-setup.jpg)

具体使用可以参考https://github.com/libfuse/sshfs



#### 4.4 使用NFS挂载文件系统

可以选择手动挂载NFS到本地的文件系统后，启动容器来生成一些数据容器，其他的容器可以利用--volumes-from来加载文件系统，同时可以考虑的另一种方式是docker-compose在v3版本中指定的.

```
version: "3.2"

services:
  rsyslog:
    image: jumanjiman/rsyslog
    ports:
      - "514:514"
      - "514:514/udp"
    volumes:
      - type: volume
        source: example
        target: /nfs
        volume:
          nocopy: true
volumes:
  example:
    driver_opts:
      type: "nfs"
      o: "addr=10.40.0.199,nolock,soft,rw"
      device: ":/docker/example"

```

#### 4.5 容器中运行GUI程序

首先创建一个ubuntu容器，安装对应的GUI程序，这里我们使用加载一个firefox 对应的Dockerfile文件如下，记得替换对应的GID和UID以及username成系统当前的用户(id命令查询)

```
FROM ubuntu:14.04

RUN apt-get update && \
    apt-get install -y firefox

RUN groupadd -g 1000 mingkai
RUN useradd -d /home/mingkai -s /bin/bash -m mingkai -u 1000 -g 1000

USER mingkai
ENV HOME /home/mingkai
CMD /usr/bin/firefox

```

如果当前宿主机运行的ubuntu版本是18.04，需要额外的安装x11服务器

```
sudo apt install xorg
```

对上面的dockerfile使用build命令获得镜像文件firefox,然后直接启动无认证模式下的连接，注意这种模式下将允许任何人从任何的主机连接进来。

```
docker build -t firefox .
xhost +    
docker run -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=$DISPLAY firefox

➜  ~ xhost -
access control enabled, only authorized clients can connect
```

另外一种方式则是使用认证方式

```
docker run -v /tmp/.X11-unix:/tmp/.X11-unix -e DISPLAY=$DISPLAY firefox -h $HOSTNAME -v $HOME/.Xauthority:/home/$USER/.Xauthority 
```



具体的可以参考: [xorg](https://www.x.org/wiki/)

#### 4.4 检查容器信息

检查某一个容器的信息的时候，可以使用对应inspect 获取其中的格式化输出。

```
docker inspect --format '\{\{.NetworkSettings.IPAddress\}\}' 容器标示
```

如果要对于多个信息进行操作，比如下面检查所有活跃的容器并进行ping操作

```
docker ps -q | xargs docker inspect --format '\{\{.NetworkSettings.IPAddress\}\}' | xargs ping -c2
```



> 尽量使用docker stop关闭容器，stop发送TERM信号，程序会接收到执行一些清理工作，docker kill 发送的是 KILL信号，导致程序直接退出。命令kill -9 发送KILL信号， 如果是kill 则默认是 15 也就是标准的TERM信号。



使用Docker-Machine可以管理多个宿主机上的docker服务，比如openstack, azure, digital ocean都提供了相关的配置用于连接和管理容器。

#### 4.5 构建镜像

使用ADD传输压缩文件到指定文件夹后，文件被自动解压处理。但是如果传递给ADD的是一个链接文件下载URL，则下载后的文件不会自动解压。

```dockerfile
FROM debian
RUN mkdir -p /opt/script
ADD hello.tar.gz /opt/script/
# ADD http://.../../hello.tar.gz /opt/script/ 不会被自动解压
RUN ls -lRt /opt/script
```



使用docker build的时候假如命令没有发生变化的时候，重新构建镜像往往会使用缓存，为了防止问题的发生，可以直接使用下面的命令重新构建. 输出的每一个中间层都和之前的不同

```shell
 docker build --no-cache .
```

另外的情况是，当我们在构建中如果出现问题，一直使用缓存将导致问题不断的出现，使用no-cache尝试重新构建可能会发现问题。



#### 4.6 清理容器

删除宿主机器上的所有容器, 可以将其作为alias命令加入系统环境，但是要注意很多情况下我们启动的容器仅仅是一些数据容器，这些也会被直接清理掉，所以执行命令要小心。

```
docker ps -q -a | xargs --no-run-if-empty docker rm -f 
```

仅删除已经退出的容器

```
➜  CH-4 docker run -d ubuntu sleep 1000
18cfbcb04bc5cc0cbc17930031678c1cea5aeaa8074ef33d9f4b37a4c5b0660d

➜  CH-4 docker ps -q -a --filter status=exited | xargs --no-run-if-empty docker rm -f 
➜  CH-4 docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
18cfbcb04bc5        ubuntu              "sleep 1000"        33 seconds ago      Up 32 seconds                           epic_haibt
```
 删除正在运行的容器
```
➜  CH-4 docker ps -q -a --filter status=running | xargs --no-run-if-empty docker rm -f 
18cfbcb04bc5

```



对于孤立的数据容器管理则稍微麻烦一些，比如容器A为一个数据容器,B是一个挂载了A的应用容器，当A被删除的时候，数据部分则由守护进程进行管理。当两个容器都被删除的时候，数据仍旧残留在系统中。

```
docker run -v /shared-folder ubuntu /bin/true  #创建数据容器A
docker run -d --volumes-from cb52ba476e4e ubuntu sleep 1000   #挂载数据容器A到B
docker rm -f cb52ba476e4e   # 删除数据容器A
```

方法是挂载一个专门的清理脚本的容器，具体可以参考[cleanup-volumes](https://github.com/docker-in-practice/docker-cleanup-volumes)

```
docker run -v /var/run/docker.sock:/var/run/docker.sock -v /var/lib/docker:/var/lib/docker --rm martin/docker-cleanup-volumes --dry-run
```

dry-run选项允许仅仅打印可能会被删除的，而不是真正的执行删除，去除掉后重新执行可以直接删除

#### 4.7 常用操作

接触当前对于容器的连接，防止直接退出后容器也随着退出。使用attach重新绑定连接

```shell
➜  docker docker run -it ubuntu /bin/bash
root@1f34ee5893a1:/# %     #使用ctrl+p 之后按ctrl+q                                       ➜  docker docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
1f34ee5893a1        ubuntu              "/bin/bash"         12 seconds ago      Up 11 seconds                           boring_tereshkova
➜  docker docker attach 1f
root@1f34ee5893a1:/# 

```



使用docker exec 命令来管理直接执行的命令，可以在已经启动的容器上执行，或者进入到容器内部执行。

```shell
➜  docker docker run -d ubuntu sleep 1000
ca13f6e27bbaa62934f1d335444bb6e1c9b8f6ef2b53f65cb6c7d40aee4e2ebd
➜  docker 
➜  docker 
➜  docker docker exec ca echo "hello" 
hello
➜  docker docker exec -it ca /bin/bash
root@ca13f6e27bba:/# 

```

### 5. 配置管理

下面的例子使用entrypoint来指定容器启动时候运行的脚本，而CMD则指定传递的参数。

```
FROM ubuntu:14.04
ADD clean_log /usr/bin/clean_log
RUN chmod +x /usr/bin/clean_log

ENTRYPOINT [ "/usr/bin/clean_log" ]
CMD [ "7" ]
```

```
➜  CH-5 docker run -v /var/log/apt/:/log_dir clean-log 365
start to clean log before 365 days
```

这样我们就可以使用这个容器来定期的清理日志数据，并赋予执行者一定的控制权限。

如果我们需要对于文本进行修改，则除了使用sed命令外还可以使用perl来修改, 使用起来更简单 ，且比较容易记忆，同时支持处理多个文件的操作只需要使用*替换后面的文件名称即可。

```
perl -p -i -e "s/#echo/echo/g" clean_log.sh
```

#### 5.1 构建历史

使用docker history可以查看整个镜像的构建历史，比如上面我们生成的clean-log镜像，但是如果构建步骤中一些操作我们不想让别人看到的话，则可以使用docker import 和docker export来实现。docker export 会将容器打包到一个tar文件，而docker import则导入镜像并设置名称。具体的可以参考docker[官方文档](https://docs.docker.com). 

```
➜  CH-5 docker history clean-log
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
2b28966a01dc        10 minutes ago      /bin/sh -c #(nop)  CMD ["7"]                    0B                  
4591828e265c        10 minutes ago      /bin/sh -c #(nop)  ENTRYPOINT ["/usr/bin/cle…   0B                  
8d14a76df334        10 minutes ago      /bin/sh -c chmod +x /usr/bin/clean_log          109B                
edb2fb1ea38b        10 minutes ago      /bin/sh -c #(nop) ADD file:968c49485c7a2e47b…   109B                
f216cfb59484        4 weeks ago         /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
<missing>           4 weeks ago         /bin/sh -c mkdir -p /run/systemd && echo 'do…   7B                  
<missing>           4 weeks ago         /bin/sh -c rm -rf /var/lib/apt/lists/*          0B                  
<missing>           4 weeks ago         /bin/sh -c set -xe   && echo '#!/bin/sh' > /…   195kB               
<missing>           4 weeks ago         /bin/sh -c #(nop) ADD file:ecefeeae93e44cb42…   188MB 
```



#### 5.2 还原Dockerfile

如果有一个镜像本身的dockerfile已经找不到了，如何通过镜像的方式获得原本的dockerfile? 方式可以有多个，可以通过docker history查看历史，可以借助于docker inspect来找到其中每一层的一些执行信息。或者是通过一个容器工具来实现。

```
docker run -v /var/run/docker.sock:/var/run/docker.sock \
  centurylink/dockerfile-from-image <IMAGE_TAG_OR_ID>
```

以下是docker inpect查看其中某一层的实例，可以使用jq工具来获得输出结果

```
$ docker inspect 011aa33ba92b
[{
  . . .
  "ContainerConfig": {
    "Cmd": [
        "/bin/sh",
        "-c",
        "#(nop) ONBUILD RUN [ ! -e Gemfile ] || bundle install --system"
    ],
    . . .
}
```

#### 5.3 Makefile与Docker

下面的Makefile执行简单的版本管理，如果用户在输入make的时候指定编译版本，则将版本同步到docker环境变量中.

```makefile
VERSION ?= v1.0

base:
	sed -i 's/{BUILD_VERSION}/${VERSION}/' Dockerfile
	docker build -t app .
	sed -i 's/${VERSION}/{BUILD_VERSION}/' Dockerfile    #替换回原来的变量名称

```

Dockerfile中则仅仅是显示和打印该版本

```dockerfile
FROM ubuntu:14.04
ENV VERSION={BUILD_VERSION}                             #设置环境变量
CMD ["sh", "-c", "echo $VERSION" ]
```

下面的脚本是获取容器中指定文件或者文件夹内容的makefile命令, 使用变量GETFILE ?= 来设置对应的文件，通过docker run --rm 执行后容器直接自动删除。

```
singlefile:
	docker run --rm app cat ${GETFILE} > outfile
multifile:
	docker run --rm -v ${pwd}/output/: /out app sh -c "cp -r ${GETMULTIFILES} /out/" 
```



#### 5.4 源代码到镜像

基于上面makefile我们可以自己写一些代码到镜像的自动化工具，但是现在有一些开源的项目可以帮助完成自动化的步骤。比如OpenShift的[source-to-image](https://github.com/openshift/source-to-image)工具.

首先下载对应的二进制工具

https://github.com/openshift/source-to-image/releases/download/v1.1.12/source-to-image-v1.1.12-2a783420-linux-amd64.tar.gz

使用该工具的主要步骤如下：

1. create 创建一个框架使用```s2i create 框架生成的Dockerfile镜像 将来的程序镜像名称```
2. 修改Dockerfile
3. 修改s2i/bin/下的assemble，run等脚本，并增加必要的配置和源代码 
4. 执行make命令并生成对应的镜像**builder image**
5. 执行测试程序 make test

具体的配置可以参考[create-s2i-builder-image](https://blog.openshift.com/create-s2i-builder-image/)



#### 5.5 镜像的维护

##### 5.5.1 使用更小的镜像

有两个比较小的基础镜像可以供选择，一个是BusyBox，另外一个是Alpine，很多镜像都是基于这两个镜像构建的，提供基础的功能，但是Alpine的包管理功能更好一些。BusyBox和Alpine的基础镜像大小大约只有不到3MB的空间占用. 下面的这个应用实际占用约30MB左右。

```
FROM gliderlabs/alpine:3.1
RUN apk-install mysql-client
ENTRYPOINT [ "mysql" ]
```

#### 5.5.2 合并执行的命令

下面的容器中执行了很多命令，每一个RUN命令都会创建一个新的分层，后面的命令将在此基础上进行复制修改，如果我们可以创建一个命令的话，则仅仅保持一层，数据会占用明显减少。

```
RUN apt-get update 
RUN apt-get upgrade -y
RUN apt-get install -y wget
RUN apt-get install -y git
RUN apt-get install -y python-pip python-dev build-essential 
RUN git clone https://github.com/zhangmingkai4315/ContactCli.git
WORKDIR /ContactCli
RUN pip install -r requirements.txt
RUN nose
```

我们可以使用符号连接将上面的命令转换为一个

```
RUN apt-get update && \
    apt-get upgrade -y && \ 
    apt-get install -y wget && \ 
    apt-get install -y git && \ 
    apt-get install -y python-pip python-dev build-essential  && \ 
    git clone https://github.com/zhangmingkai4315/ContactCli.git
```

或者我们将整个的执行写入一个脚本来完成，同时还可以进入容器进行删减去除不必要的doc（share）文件等，并利用扁平化的方式（import和export）来压缩空间占用。



##### 5.5.3 直接使用编译好的文件

将编译好的文件直接拷贝到容器运行，减少编译环境，可以大幅度的减少容器的大小，且减少dockerfile的流程.

使用inotifywait可以查看一个程序运行时访问的文件列表，通过删除一些不需要的文件也可以使得容器变得很小。

但实际操作的时候不能表现的足够完美，很难发现所有的文件，有可能某些场景会使用该文件但是却被删除了。



#### 5.6 企业镜像的管理

如果仅仅考虑构建一些小的镜像，由于不同应用之间需要的依赖不同，很有可能需要很多不同类型，不同应用场景的基础镜像，实际占用的资源可能会更多。

作为一个企业，应该考虑更多的是构建一个足够大小的基础镜像，这个镜像需要经过充分测试，作为所有镜像的依赖镜像，由于Dockerfile的分层构建，这个镜像会被所有的所共享，也就是资源上使用更少，管理上也更方便。