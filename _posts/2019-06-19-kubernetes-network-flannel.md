---
layout: post
title: Kubernetes网络及组件Flannel实施
date: 2019-06-19T11:32:41+00:00
author: mingkai
permalink: /2019/06/kubernetes-network-flannel
categories:
  - kubernetes
  - docker
---

## Kubernetes网络及组件Flannel实施

### 1. Kubernetes网络模型

对于Kubernetes的网络模型，默认情况下本身并不关心底层网络是如何通信的，Kubernetes使用Pod进行容器的管理，每个Pod中都有不同数量的容器运行，同一个Pod上的不同容器共享相同的网络协议栈，可以直接通过Localhost进行通信。



![](https://cdn-images-1.medium.com/max/1600/1*L7dDIpRo8x91gEegGvDohg.png)



Kubernetes网络模型要求：

- 所有容器均可以不通过NAT的情况下直接进行网络通信
- 所有的主机可以与所有的容器（反之也可以）进行网路通信
- 容器本身看到的IP和其他容器看到的IP相同

网络模型默认所有的容器都已经可以正常的进行网络传输。但是实际上运行在不同node上的容器是不能直接进行通信，因此需要通过一些辅助的网络组件完成数据的传输。

### 2. Overlay网络

Flannel是由CoreOS开发并维护的一个Kubernetes网络组件，主要的职责是创建一个不分层的平整的网络模型，在这个模型里面所有的主机都被分配一个独立的IP地址，并可以通过该地址进行彼此的通信。如下图所示：

![](https://cdn-images-1.medium.com/max/1600/1*EFr8ohzABfStS7o9gGMYKw.png)

在上面的图示中，底层的主机之间通过一个172.20.32.0/19的网络进行通信， flannel为每一个节点的docker容器组分配一个独立的网段比如第一个100.96.1.0/16，网关为一个docker0的地址。所有分配的容器均位于该网段内。集群所有的容器分配的地址均属于一个大的网段： 100.96.0.0/16

同一台机器中的容器可以直接的通过docker0进行数据通信。而不同机器上的容器则必须通过路由表和UDP封装的形式实现。

为了实现不同主机之间的容器通信，需要进行多次数据包封装和传递才可以完成，如下图所示

![](https://cdn-images-1.medium.com/max/1600/1*JqSLd3cPv14BWDtE7YEcRA.png)



在每一个主机上flannel运行一个独立的进程flanneld，用于创建路由表

```shell
admin@ip-172-20-33-102:~$ ip route
default via 172.20.32.1 dev eth0
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.1.0
100.96.1.0/24 dev docker0  proto kernel  scope link  src 100.96.1.1
172.20.32.0/19 dev eth0  proto kernel  scope link  src 172.20.33.102
```

对于从100.96.1.2到达100.96.2.3的数据包，会处于其中第二条路由规则，经过flannel0网卡，flannel0网卡是一个由flanneld创建的TUN设备(在内核和用户程序中直接传递IP数据)

- 当写IP数据的时候，数据包直接发往内核，内核根据路由表进行数据路由
- 当接受数据的时候，路由表将数据包直接从内核发送到flannel0对应的程序flanneld中

当接收到数据后flanneld需要知道该数据包到底要传递到哪一台机器上去。flannel需要借助于etcd去存储这些数据，并缓存在内存中。

```shell
etcdctl ls /coreos.com/network/subnets
/coreos.com/network/subnets/100.96.1.0-24
/coreos.com/network/subnets/100.96.2.0-24
/coreos.com/network/subnets/100.96.3.0-24
```

```
etcdctl get /coreos.com/network/subnets/100.96.2.0-24
{"PublicIP":"172.20.54.98"

#扩展命令
$ etcdctl -o extended get /coreos.com/network/subnets/10.5.34.0-24
Key: /coreos.com/network/subnets/10.5.34.0-24
Created-Index: 5
Modified-Index: 5
TTL: 85925
Index: 6

{"PublicIP":"10.37.7.195","BackendType":"vxlan","BackendData":{"VtepMAC":"82:4b:b6:2f:54:4e"}}
```

通过查询既可以得到具体的数据包传递的， 因此flanneld将数据包封装在UDP中，传递出去。

数据发送后，对端的flanneld获得数据包，根据路由表分配到docker0所在的网关中，通过网关传递到对应的容器。

在flanneld的配置中使用下面的环境变量来管理内容， 同时该配置文件也被用于dockerd的启动配置

```
# cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```

```
dockerd --bip=$FLANNEL_SUBNET --mtu=$FLANNEL_MTU
```

新版本中的flannel不推荐使用UDP来封装数据，同时flannel0的TUN设备的直接拷贝的方式传递数据包到内核，和从内核获得数据包也存在性能的损耗，因此线上配置的时候我们一般通过设置其他的方式比如vxlan的来传递数据包。

### 3. 安装Flannel

#### 1. 使用kubernetes yaml安装

项目Github： <https://github.com/coreos/flannel>

从<https://github.com/containernetworking/plugins/releases>获取最新的网络插件二进制程序包，并解压到每一台node的/opt/cni/bin目录

设置kubelet的启动参数,增加启动参数并重启

```shell
 --network-plugin=cni --cni-bin-dir=/opt/bin,/opt/cni/bin
```

设置kube-controller-manager的启动参数, 增加并重启

```shell
--allocate-node-cidrs=true --cluster-cidr=10.244.0.0/16
```

下载flannel的配置yaml文件，并使用kubectl启动: 

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

运行完成后 检查对应的配置文件```/run/flannel/subnet.env```是否存在, 如果存在的话

修改对应docker的systemd文件，配置环境变量和启动方式如下：

```
# egrep -v '^(;|#|//|$)' /lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service
Wants=network-online.target
Requires=docker.socket
[Service]
EnvironmentFile=-/run/flannel/subnet.env
Type=notify
ExecStart=/usr/bin/dockerd --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
```

完成后重启docker和kubelet即可。

这种方式下，存在的一个问题是如果机器重启，则由于尚未部署所需要的flannel容器，无法生成对应的env文件，docker启动的时候无法获得配置文件导致启动失败。可以考虑环境变量重载的方式，写入一个缺省的环境变量到服务启动文件中即可。

#### 2. 本地直接安装

直接安装的方式很简单，就像kubelet和kube-proxy一样，下载官方的release文件，内部包含两个关键的文件，一个是flanneld，另一个mk-docker-opts.sh ， 将两个文件复制到/usr/bin下即可

创建flannel对应的systemd文件

```yaml
[Unit]
Description=flannel
Documentation=https://github.com/coreos/flannel
After=network.service
Requires=docker.service

[Service]
Type=notify
EnvironmentFile=-/etc/sysconfig/flannel
ExecStart=/usr/bin/flanneld $FLANNEL_ETCD $FLANNEL_ARGS
ExecStartPost=/usr/bin/mk-docker-opts.sh -c

[Install]
RequiredBy=docker.service
WantedBy=multi-user.target

```

其中Post执行的语句mk-docker-opts.sh用于生成对应的环境变量到docker的启动环境文件中，默认的为/run/docker_opts.env

其中/etc/sysconfig/flannel配置文件如下, 由于是直接安装的方式，所以我们连接etcd即可，不需要再通过kube-apiserver查询。

```
FLANNEL_ETCD="-etcd-endpoints={{ etcd_servers }}"
FLANNEL_ARGS="--ip-masq=true"
```



配置docker的启动文件确保对应的环境文件已经加入进去，并重启docker.

```
EnvironmentFile=/run/docker_opts.env
ExecStart=/usr/bin/dockerd $DOCKER_OPTS
```

使用这种方式，无需再kubelet启动的时候增加cni参数和对应的cni-bin查询位置参数

#### 相关问题

 **问题1：**如果出现以下的错误，确认/etc/cni/net.d/文件下面是否存在其他的网路配置文件，如果有则删除，并删除对应的cni网卡\```ip link delete cni0``` 重启kubelet和docker容器

```
NetworkPlugin cni failed to set up pod "busybox-2_default" network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.2.1/24
```

对于仍旧存在问题，则考虑重新在节点上生成对应的配置

```
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
systemctl restart kubelet
systemctl restart docker
```



**问题2：** 无法实施不同主机上的pod的通信？

确保flanneld启动参数包含```-ip-masq```参数



