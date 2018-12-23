---
layout: post
title: Kubernetes节点与资源管理
date: 2018-12-20T11:32:41+00:00
author: mingkai
permalink: /2018/12/kubernetes-node-and-resource
categories:
  - kubernetes
  - docker
---

### 1. Node管理

#### 1.1 隔离Node

如果一个node需要隔离，进行安全加固或者硬件升级，可以对于该node进行隔离操作，使其暂时不接受Pods的调度，如下为三台机器上面运行了很多的容器，现在需要对于其中的192.168.0.201进行隔离操作。

```shell
$ kubectl get nodes
NAME            STATUS   ROLES    AGE   VERSION
192.168.0.201   Ready    <none>   11d   v1.13.0
192.168.0.202   Ready    <none>   11d   v1.13.0
192.168.0.203   Ready    <none>   11d   v1.13.0
```

具体的执行可以直接使用命令行进行， 对于其中的节点进行drain操作，这种方式可以保证其系统每终止一个Pod则在另外一个健康的Node上面创建一个新的Pod, 然后继续完成其他的终止操作。 有时候如果定义了PodDisruptionBudget则可能不允许进行主动的驱逐，需要删除后进行操作。

如果该节点上存在临时性的存储对象，需要确认是否可以删除，比如临时的日志数据等，这里我们对于临时的存储对象进行强制删除。

```shell
$ kubectl drain 192.168.0.201
node/192.168.0.201 cordoned
error: unable to drain node "192.168.0.201", aborting command...

There are pending nodes to be drained:
 192.168.0.201
error: pods with local storage (use --delete-local-data to override): shared-storage-app-s9cv6

$ kubectl drain 192.168.0.201 --delete-local-data
```

再次检查Pods的状态会发现该节点上已经不再包含任何的Pods, 节点的状态也变成了SchedulingDisabled， 无法被调度的状态，实现了真正的隔离，这时候可以进行安全的升级操作等。

```shell
$ kubectl get nodes
NAME            STATUS                     ROLES    AGE   VERSION
192.168.0.201   Ready,SchedulingDisabled   <none>   11d   v1.13.0
192.168.0.202   Ready                      <none>   11d   v1.13.0
192.168.0.203   Ready                      <none>   11d   v1.13.0
```

如果系统已经升级完毕，记得对其进行重新的恢复操作，执行下面的命令可以实现恢复调度操作， 这时候201节点又可以重新的执行新的Pods了。

```sh
$ kubectl uncordon 192.168.0.201
node/192.168.0.201 uncordoned
```



#### 1.2 扩容Node

扩容Node，仅仅需要在确保网络联通的情况下， 配置新的节点的Docker,kubelet和kube-proxy使其能够正常的启动后联通kubernetes master即可，确保启动参数的正确，如果设置了认证的话，需要拷贝对应的证书文件等到相关的node上保证认证的有效。

#### 1.3 Node的标签管理

利用Node的标签可以对于pod进行特殊的节点的指定运行，比如下面的201节点，新增了一个service=database的标签，我们可以在调度Pod的时候，指定其调度到该机器上（但是一旦该机器出现问题，可能由于没有可用的标签节点而无法自动的创建）

```sh
$ kubectl get nodes --show-labels
NAME            STATUS   ROLES    AGE   VERSION   LABELS
192.168.0.201   Ready    <none>   11d   v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.0.201,service=database
192.168.0.202   Ready    <none>   11d   v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.0.202
192.168.0.203   Ready    <none>   11d   v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.0.203
```

创建标签的操作可以使用下面的命令, 为了确保可以调度成功，我们可以在另外一台机器上也创建相同的标签，如下面所示:

```sh
$ kubectl label node 192.168.0.202 service=database
node/192.168.0.202 labeled
```

删除标签和覆盖标签的操作如下, 覆盖标签的时候需要添加--overwrite来完成，否则无法直接设置：

```
mike@PV-X00189105:~$ kubectl label node 192.168.0.202 service-
node/192.168.0.202 labeled

mike@PV-X00189105:~$ kubectl label node 192.168.0.201 service=dns --overwrite
node/192.168.0.201 labeled

```



### 2. 集群Namespace

kubernetes集群的namespace可以用来划分逻辑的分区给不同的部门或者业务使用, 确保既可以共享相同的硬件资源，又不会互相产生干扰。

![](http://static1.tothenew.com/blog/wp-content/uploads/2016/12/kube.png)

创建namespace的方式可以通过yaml文件来完成

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
```



创建完成后查看既可以看到刚刚创建的namespace信息：

```sh
$ kubectl get namespace
NAME          STATUS   AGE
default       Active   13d
development   Active   12s
kube-public   Active   13d
kube-system   Active   13d
production    Active   8s
```



修改当前kubectl使用的namspace信息：

```shell
mike@PV-X00189105:/mnt/d/kubernetes/workspace$ kubectl config set-context $(kubectl config current-context) --namespace=development
Context "default-context" modified.

mike@PV-X00189105:/mnt/d/kubernetes/workspace$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: http://192.168.0.200:8080
  name: development
contexts:
- context:
    cluster: development
    namespace: development
    user: cluster-admin
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users:
- name: cluster-admin
  user:
    password: "******"
    username: mike
```

这时候查看pod, rc或者service均为空，因为我们并没有在这个新配置的namspace下面新建任何的资源信息，这也确保了隔离性，其他空间下的资源我们也没有办法查看。



### 3. 资源分配



下面一个是针对RC的资源配置文件，这里我们指定了容器的资源消耗，并设置了两类资源限制

- requests: 定义容器希望分配到的，可以完全保证的资源量
- limits: 定义容器最多使用的资源量

另外需要确保资源的使用量是否可以允许进程正常的工作，比如下面的mysql如果设置低于512Mi则可能无法正常的启动应用。

```yaml
apiVersion: v1
kind: ReplicationController
metadata: 
  name: mysql-db
spec: 
  replicas: 1
  selector:
    app: mysql
  template:
    metadata:
      name: mysql-db
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql-db
        image: mysql
        resources:
          requests: 
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

当启动Pod的时候，Kubernetes将Request和Limit值转换为容器启动参数传递给容器执行，比如docker容器的话则传递的是--cpu-share，--cpu-quota --cpu-period 以及--memory等等

运行起来后，检查所在的node的详细信息可以看到定义的内存和CPU占总量的多少。

```
  $ kubectl describe nodes 192.168.0.202
  Namespace                  Name              CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----              ------------  ----------  ---------------  -------------  ---
  development                mysql-db-kh8rj    250m (25%)    500m (50%)  512Mi (58%)      1Gi (117%)     10m
```

这里我们设置了最大的内存占用为1Gi，但是本身节点的内存都无法达到该最大的内存限制，也就没有任何的意义。为了保证在声明的时候就不超过最大的部署内存大小，我们可以定义资源的限制和缺省配置, 下面是创建一个资源的限制，创建后，每一个新的资源在创建的时候都必须

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: computer-resources
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
```

我们修改之前的mysql rc资源的使用量

```yaml
apiVersion: v1
kind: ReplicationController
metadata: 
  name: mysql-db
spec: 
  replicas: 2
  selector:
    app: mysql
  template:
    metadata:
      name: mysql-db
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql-db
        image: mysql
        ports:
          - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "admin"
        resources:
          requests: 
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
    
```

检查当前的资源使用量：

```yaml
$ kubectl describe resourcequota computer-resources
Name:            computer-resources
Namespace:       development
Resource         Used   Hard
--------         ----   ----
limits.cpu       500m   2
limits.memory    1Gi    800Mi
requests.cpu     250m   1
requests.memory  512Mi  256Mi
```

如果我们创建的服务中没有包含对应的资源使用配置，则无法正常的启动,错误信息如下：

```
  Warning  FailedCreate  13s               replication-controller  Error creating: pods "nginx-app-s2l9n" is forbidden: failed quota: computer-resources: must specify limits.cpu,limits.memory,requests.cpu,requests.memory
```

为了解决这个问题，我们可以设置缺省的资源使用:

```yaml
apiVersion: v1
kind: LimitRange
metadata: 
  name: limits
spec:
  limits:
  - default:
      cpu: 200m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 256Mi
    type: Container
```

创建后再次使用没有配置资源的文件进行创建，则会默认的分配指定大小的资源使用量：

```yaml
    $ kubectl describe pods nginx-app-spd9r
    Limits:
      cpu:     200m
      memory:  512Mi
    Requests:
      cpu:        100m
      memory:     256Mi
    Environment:  <none>
```

重新检查当前命名空间下面的资源使用率：

```yaml
$  kubectl describe resourcequota computer-resources
Name:            computer-resources
Namespace:       development
Resource         Used   Hard
--------         ----   ----
limits.cpu       1400m  4
limits.memory    2Gi    8Gi
requests.cpu     700m   2
requests.memory  1Gi    2Gi
```

> 配额的动态修改不会影响已经创建的资源的使用情况。当集群中总的资源小于命名空间中资源的总和，则导致资源竞争，按照先到先得的方式进行配置

### 4. 资源服务质量QoS

Kubernetes根据定义的配置中是否声明了Request和Limit来确定该服务的服务质量（QoS）, 对于一些高可靠性要求的Pod或者服务在声明的时候就需要申请可靠的资源。 对于Request=Limit的情况，代表了所申请的资源都是完全可靠的资源，而对于Request<Limit的则其中包含Request大小可靠的资源，以及Limit-Request的不可靠资源。

#### 4.1 资源分类

- 可压缩资源: 当前的CPU属于可压缩资源，并且按照Pod创建时候的Request值来进行分配，多余的CPU资源也会按照其Request比例进行分配。
- 不可压缩资源： 当前的内存属于不可压缩资源，如果内存使用量小于Request的量，则Pod可以正常工作，如果超出的话，则Pod可能会被杀掉。如果Pod使用内存超过了Limits的值，则操作系统内核会杀掉Pod所有容器的所有进程中内存最多的一个，使其恢复Limit范围内。

#### 4.2 服务质量分类

QoS级别直接有Limit和Request来定义，下面的分别是三种不同的服务质量的定义：

**1. Guaranteed完全可靠**： Pod中所有容器均定义了limits和request，且其中的Limits和Requests包含的内存和CPU的数值相同，这样的资源下，不会因为超出使用范围而被杀掉。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```



对于未定义Request的情况，如果定义了对应的Limit值则其**自动设置Request值为Limit的值**



**2. Burstable弹性保证**

至少Pod中的一个容器包含一个内存或者CPU的Request定义，容器如果未定义Limit则默认的Limit为节点的资源上限。优先级居中，当没有Best-Effort(最低级别)的容器可供清理的话，则Pod的进程可能被杀死。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-2
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
```



**3. BestEffort尽力保证**

这种保证的级别最小，资源声明的时候未定义任何的内存和CPU的Request值和Limit值。资源充足的时候可充分的使用限制资源，但是一旦资源紧张，则可能在第一批就被杀掉，且资源消耗最多的Pod被首先驱逐。

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-3
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-3-ctr
    image: nginx
```



当Pod的CPU资源Requests无法得到满足的时候，容器的CPU会被压缩限流，而内存由于是无法压缩的所以在内存紧张的时候按照QoS的级别进行处理。另外Kubernetes内部使用OOM积分系统进行计算来完成资源紧张时候的容器清理。



当节点的资源不足的时候，节点会向Master汇报，不再继续向该节点调度新的Pod. 另外最佳实践的话， 设置kubelet的启动参数，设置驱逐限制，防止Pod的资源占用太多系统资源，导致系统的OOM。

