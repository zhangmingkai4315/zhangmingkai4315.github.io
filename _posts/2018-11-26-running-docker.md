---
layout: post
title: Docker运行方面的一些问题
date: 2018-11-26T11:32:41+00:00
author: mingkai
permalink: /2018/11/running-docker
categories:
  - docker
  - devops
---

### 1. 安全隔离

对于docker客户端，由于内部的api调用是需要root权限的，所以客户端需要使用sudo命令执行操作。当一个用户可以在主机上运行docker的时候，其生成的容器的root权限其实在某些方面和外面的root用户相差不大。比如查看当前宿主机系统的root权限文件资源。

```
docker run --rm -v /etc/shadow:/etc/shadow ubuntu:14.04 cat /etc/shadow
root:!:17833:0:99999:7:::
daemon:*:17737:0:99999:7:::
bin:*:17737:0:99999:7:::
sys:*:17737:0:99999:7:::
sync:*:17737:0:99999:7:::

```

当使用docker run的时候，docker会创建一系列的命名空间和cgroups，docker利用linux的命名空间实现了对于资源的隔离，比如进程命名空间使得容器只能看到和容器有关的进程，其他进程是无法查看到的。网络空间则使得容器具有自己独特的网络栈，与其他容器进行通信，就好像运行的两个主机进行通信一样。

但是操作系统不是所有部分都使用命名空间，比如设备资源就没有使用命名空间。默认情况下docker运行容器是**unprivileged**的状态，因此不能访问任何的设备文件，也就不能在一个容器内部再运行一个docker进程，但是一旦执行的时候加入```--privileged```的参数，则当前的容器既可以访问任何的宿主机设备资源，就如同直接运行在容器外部的进程一样可以使用这些资源。

可以设置**device**参数来限制仅仅访问那些设备资源比如下面的命令：

```
$ docker run --device=/dev/snd:/dev/snd ...
```

容器本身缺省对于这些设备资源具有read,write和mknod的权限，如下面命令，同时我们可以设置对于资源的操作仅仅使用其中的某一些权限。

```
$ docker run --device=/dev/sda:/dev/xvdc --rm -it ubuntu fdisk  /dev/xvdc

Command (m for help): q
$ docker run --device=/dev/sda:/dev/xvdc:r --rm -it ubuntu fdisk  /dev/xvdc
You will not be able to write the partition table.

Command (m for help): q

$ docker run --device=/dev/sda:/dev/xvdc:w --rm -it ubuntu fdisk  /dev/xvdc
    crash....

$ docker run --device=/dev/sda:/dev/xvdc:m --rm -it ubuntu fdisk  /dev/xvdc
fdisk: unable to open /dev/xvdc: Operation not permitted

```



cgroups是linux容器的另一个重要部分，实施资源的分配和限制，并且确保单一的容器不会消耗大量的资源而导致系统的崩溃, 但是cgroups并不能防止容器去访问其他容器的资源。

如果在宿主机上启动了多个容器，则多个root用户的操作必然会带来一些安全隐患，一些企业在使用的时候给每一个用户提供一台虚拟机，将同一个用户的容器放置在一台虚拟机上进行安全隔离，缓解安全隔离的问题。

linux内核在设计的时候将单一的root权限划分位可以独立授予的功能权限，比如下面的表中所列的一部分，同时docker开发的时候也默认关闭了一些功能，因此及时容器中的root有些事也是做不了的，比如CAP_NET_ADMIN这种管理整个网络栈的能力是默认关闭的，而CAP_NET_BIND_SERVICE则是开启的，下面是通过```man 7 capabilities```命令查看到的部分功能授权：

```shell
       CAP_NET_ADMIN
              Perform various network-related operations:
              * interface configuration;
              * administration of IP firewall, masquerading, and accounting;
              * modify routing tables;
              * bind to any address for transparent proxying;
              * set type-of-service (TOS)
              * clear driver statistics;
              * set promiscuous mode;
              * enabling multicasting;
              * use  setsockopt(2)  to  set  the  following  socket options:
                SO_DEBUG, SO_MARK, SO_PRIORITY (for a priority  outside  the
                range 0 to 6), SO_RCVBUFFORCE, and SO_SNDBUFFORCE.

       CAP_NET_BIND_SERVICE
              Bind  a  socket to Internet domain privileged ports (port num‐
              bers less than 1024).

       CAP_NET_BROADCAST
              (Unused)  Make socket broadcasts, and listen to multicasts.

       CAP_NET_RAW
              * Use RAW and PACKET sockets;
              * bind to any address for transparent proxying.

```

尽管可以设置```--cap-drop```来关闭某些功能，但是root本身仍旧具有操作自身文件的能力，比如最开始的命令仍旧是有效的, 取消掉的大部分是管理其他用户相关的功能，而不是root本身的功能。

```
docker run --rm --cap-drop=all -v /etc/shadow:/etc/shadow ubuntu:14.04 cat /etc/shadow
root:!:17833:0:99999:7:::
daemon:*:17737:0:99999:7:::
bin:*:17737:0:99999:7:::
sys:*:17737:0:99999:7:::
sync:*:17737:0:99999:7:::
games:*:17737:0:99999:7:::
man:*:17737:0:99999:7:::
lp:*:17737:0:99999:7:::
mail:*:17737:0:99999:7:::
news:*:17737:0:99999
```

###  2. 监控资源使用

使用cAdvisor可以实时查看当前资源的使用情况，同时借助于prometheus等工具可以很好的集成到企业现有的监控系统中。下面是在本地宿主机上运行一个cAdvisor的实例。

```
docker run --volume /:/rootfs:ro --volume /var/run:/var/run:rw --volume /sys:/sys:ro --volume /var/lib/docker/:/var/lib/docker:ro -p 8080:8080 -d --name cadvisor --restart on-failure:3 google/cadvisor

```

实时输出的信息包括：

- 网络资源使用
- 系统资源使用
- 容器隔离情况
- 历史资源使用等

### 3. 控制资源使用

#### 3.1 限制内存使用

如果在ubuntu上启动的docker可能出现如下的提示, 代表当前无法限制内存使用。

```
docker info | grep swap
WARNING: No swap limit support
```

可以进行适当的配置进行修改,具体步骤如下：

```
vi /etc/default/grub

GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"

sudo update-grub
```



利用-m参数可以设置当前程序运行限制的内存大小。最小设置是４mb

```
docker run -d -p 8081:80 --memory="256m" nginx
```

#### 3.2 CPU限制

如果一个容器中进程消耗了所有的资源，则会导致其他的容器的用户资源的匮乏，无法正常的提供服务。默认情况下，docker可以在任何内核上运行，如果容器只有一个进程和线程，则只能消耗一个内核。如果是多线程，则运行在多个内核上。下面的命令会将容器限定到第一个内核上：

```
 docker run --cpuset-cpus=0 ubuntu:14.04 sh -c 'cat /dev/zero > /dev/null'
```

我们也可以设置容器去运行在一组内核比如```0-1,3```代表运行在３个内核分别是第一个第二个和第四个

使用```-c```参数可以指定在cpu上分配的资源比例，默认1024, 下面的命令当我们仅仅执行第一条的时候，该容器会占满整个内核，但是如果我们设置第二个的时候，则该容器会平分这个CPU内核

```
docker run --cpuset-cpus=0 -c 512 ubuntu:14.04 sh -c 'cat /dev/zero > /dev/null'
docker run -it --cpuset-cpus=0 -c 512 ubuntu:14.04 bash 
root@e6026316db8e:/# ps 
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
   15 pts/0    00:00:00 ps

```

#### 3.3 文件大小限制

当使用centos系统时候，默认的docker存储驱动是devicemapper,该驱动默认的大小限制是10gb，可以通过命令docker info查看

```
vagrant@localhost ~ $ docker info
Containers: 1
 Running: 1
 Paused: 0
 Stopped: 0
Images: 8
Server Version: 1.12.1
Storage Driver: devicemapper
 Pool Name: docker-253:0-134686185-pool
 Pool Blocksize: 65.54 kB
 Base Device Size: 10.74 GB
 Backing Filesystem: xfs

```

检查任何的容器都会发现类似于如下的信息:

```
docker exec -it 24 ash
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
/dev/mapper/docker-253:0-134686185-8fc8b2dadb53e347d330f38b093cf6abaebcf008265f9e9ef6e1c2c7fb9a5d95
                         10.0G     78.6M      9.9G   1% /

```

 为了使得容器可以使用更大的存储空间，可以通过下面的方式来设置最大的容量。

```
docker daemon --storage-opt dm.basesize=20G
```

重新配置后需要重新生成容器才可以，直接运行原来的容器仍旧会沿用之前的配置参数。

#### 3.4  网络资源使用

默认情况下使用的桥接模式网络，由于中间会增加docker本身的网桥Veth绑定等，会导致网络传输效率降低约５０％左右，可以考虑使用host模式或者macvlan模式，两者基本接近于原始的传输效率

net=host模式，会导致运行的一些问题，比如暴露端口的时候，容器仅仅能够运行一个，由于都是直接暴露在主机端口上，而不是通过映射实现。

