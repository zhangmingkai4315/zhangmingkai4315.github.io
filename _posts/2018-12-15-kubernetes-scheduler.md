---
layout: post
title: kubernetes容器调度策略
date: 2018-11-26T11:32:41+00:00
author: mingkai
permalink: /2018/12/schedule-mode-in-kubernetes
categories:
  - kubernetes
  - docker
---

### 1. 测试环境

介绍基于最新版本的Kubernetes1.13完成， 使用vmware搭建运行环境，本机为windows10，可以直接运行ubuntu的子系统作为kubenetes客户端和ansible远程配置管理，其余的四台机器安装CentOS7版本的操作系统，并安装对应的Docker1.8运行环境. 具体的配置如下：

| 服务器 | IP            |安装软件|
| ------ | ------------- |----|
| master | 192.168.0.200 |kube-apiserver, kube-schedule, kube-controller-manager|
| node   | 192.168.0.201 |kubelet, kube-proxy|
| node   | 192.168.0.202 |kubelet, kube-proxy|
| node   | 192.168.0.203 |kubelet, kube-proxy|

同时服用其中的三台机器192.168.0.200、192.168.0.201、192.168.0.203部署etcd cluster，提供kv存储功能。具体的Ansible部署脚本可以参考Github上的项目[Ansible-For-Kubernetes](https://github.com/zhangmingkai4315/ansible-for-kubernetes)

### 2. 容器调度策略

Kubernetes中如果直接使用Pod的话，需要手动完成容器的部署，迁移和升级等操作，特别是当一些主机不可用的时候，直接使用Pod部署无法完成自动的迁移管理。因此为了简化日常维护的工作，kubernetes提供了多种部署方式，用来管理Pod，在线上使用的时候可以根据需要进行选择。

#### 2.1 Deployment模式

Deployment模式可以允许Kubernetes自动的管理Pod，提供持续的监控保证Pod的数量符合预期的状态。比如下面的配置文件**nginx.yaml**将启动三个运行Nginx的容器组，每个组都是一个独立的Pod，根据内部的算法选择部署的位置。Deployment创建RepicaSet来管理Pods的版本升级操作。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.4
        ports:
        - containerPort: 80
```

编写完成后使用下面的命令创建：

```sh
 kubectl create -f nginx.yaml
```

通过kubectl的get命令我们可以看到当前运行在不同节点上的Pod的信息，如下所示，当前deployment创建的三个Pod分别被放置到三台不同的主机上运行：

```sh
$ kubectl get pods -o wide
NAME                                READY   STATUS      RESTARTS   AGE     IP           NODE            NOMINATED NODE   READINESS GATES
nginx-deployment-75bd58f5c7-8tzgj   1/1     Running     0          2m36s   172.17.0.3   192.168.0.203   <none>           <none>
nginx-deployment-75bd58f5c7-9d2jq   1/1     Running     0          2m36s   172.17.0.4   192.168.0.201   <none>           <none>
nginx-deployment-75bd58f5c7-gndj7   1/1     Running     0          2m36s   172.17.0.4   192.168.0.202   <none>    
```



#### 2.2 基于节点标签的选择

kubernetes使用master服务器上的scheduler来管理容器的调度，根据内部的算法完成自动的节点选择，但是有时候我们需要一些容器运行在指定的服务器上时候，可以通过给Node打标签的方式实现。下面的命令用来显示每个nodes的标签值。kubernetes自带了一些比如系统架构，操作系统版本，主机等标签，可以直接使用无需重新定义。

```shell
$ kubectl get nodes --show-labels
NAME            STATUS   ROLES    AGE     VERSION   LABELS
192.168.0.201   Ready    <none>   4d21h   v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.0.201
192.168.0.202   Ready    <none>   4d21h   v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.0.202
192.168.0.203   Ready    <none>   4d21h   v1.13.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=192.168.0.203
```

如果我们的节点```192.168.0.201```是一台专门用来运行数据库的服务器我们可以给这台机器打上标签

```
$ kubectl label nodes 192.168.0.201 service=database
node/192.168.0.201 labeled
```

我们部署一台redis服务器用来提供数据的存储服务， 下面为配置文件, 我们创建一个具有2个复制集合的容器组，并设置nodeSelector为service=database，期望其能够部署到标记有此标签的节点上运行。

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis
  labels:
    name: redis
spec: 
  replicas: 2
  selector:
    name: redis
  template:
    metadata:
      labels:
        name: redis
    spec:
      containers:
      - name: redis-server
        image: redis
        ports:
        - containerPort: 6379
      nodeSelector:
        service: database
```

检查实际的运行情况, 当前创建了两个Pod,通过检查Pod的运行位置我们可以看到两个Pod都被分配到了相同的机器上运行

```
kubectl describe rc/redis
Name:         redis
Namespace:    default
Selector:     name=redis
Labels:       name=redis
Annotations:  <none>
Replicas:     2 current / 2 desired
Pods Status:  2 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  name=redis
  Containers:
   redis-server:
    Image:        redis
    Port:         6379/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                    Message
  ----    ------            ----  ----                    -------
  Normal  SuccessfulCreate  4m7s  replication-controller  Created pod: redis-km646
  Normal  SuccessfulCreate  4m7s  replication-controller  Created pod: redis-7h2wv
```

```sh
$ kubectl get pods -o wide --selector="name=redis"
NAME          READY   STATUS    RESTARTS   AGE    IP           NODE            NOMINATED NODE   READINESS GATES
redis-7h2wv   1/1     Running   0          7m9s   172.17.0.6   192.168.0.201   <none>           <none>
redis-km646   1/1     Running   0          7m9s   172.17.0.5   192.168.0.201   <none>           <none>
```

当我们杀掉其中一个Pod的时候，会发现重新启动的Pod依旧运行在该节点上，这就是先了通过节点来选择部署的容器的方式。

> 如果分配的nodeSelector不存在，则Pod将一直处于Pending状态，不会被成功的创建



#### 2.3 节点亲和性调度

对与节点的标签选择部署容器，方式简单直接，但是可能有些时候我们需要的是更多的调度策略，比如优先选择调度和互斥调度（不会运行在该节点上），下面先介绍的亲和性调度，提供两种参数配置

- requiredDuringSchedulingIgnoredDuringExecution强制要求的调度策略，但是会忽略已经执行的容器
- preferredDuringSchedulingIgnoredDuringExecution优先调度策略，设置权重定义执行的先后顺序

通过下面的例子可以感受下亲和性调度的模式, 注意其中的容器```k8s.gcr.io/pause```为pod中默认需要的容器，下面的会要求必须运行在amd64架构的服务器上，如果是ssd类型的磁盘最好，优先级最高。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: beta.kubernetes.io/arch
            operator: In
            values:
            - amd64
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disk
            operator: In
            values:
            - ssd
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:3.1
```

同时如果一个配置文件中同时定义了nodeSelector和nodeAffinity则必须同时满足才可以， 如果一个nodeSelectorTerms定义多个，则只需满足其中一个nodeSelectorTerms既可以，但是matchExpressions必须同时满足才可以。

#### 2.4 Pod亲和性调度

除了节点亲和度外，还可以设置Pod之间的亲和度，即如果一个Pod具有某一些标签，其他的Pod可以根据亲和度设置（比如具有相同的标签）来调度或者不去调度到相同的Node上运行。同时topologyKey用来定义匹配后的调度需要指定运行的范围是主机，集群还是Zone。

下面的文件声明了一个具有pod亲和度的Pod配置, 所有针对Pod亲和度的配置位于podAffinity下面，参数和nodeAffinity一样。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: k8s.gcr.io/pause:3.1
```

如果我们直接创建，则该Pod会一直处于Pending状态，因为当前没有符合此标签的Pod在运行我们创建一个pod来具有该标签。

```yaml
apiVersion: v1
kind: Pod
metadata: 
  name: security-pod
  labels:
    security: "S1"
    app: "nginx"
spec:
  containers:
  - name: nginx
    image: nginx
```

一旦创建完成后，之前定义了安全匹配的Pod亲和调度将立刻生效，并运行在和上面的nginxPod相同的node上。

下面的是官方给出的podAntiAffinity的实例,其中定义了一个web-server的部署集合，包含3个复制集。调度三个复制集合的时候需要确保不和任何具有app=web-store的安排在一台主机上，且必须和具有app=store的放在一个node上，同时如果app=store能够保证也有至少3个复制的，新的部署也会分散部署所有的web-server容器。



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 3
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.12-alpine
```

#### 2.5 使用Taints（污点）和Tolerations（容忍）

与上面的亲和度相反，taint和tolerations主要用来排除在一些节点上执行pod， 也是使用类似的语义来定义标签比如下面标记某一个节点一个特殊的部门标签，只有可以显式标记容忍该标签的Pod才可以在这台机器上运行。但是已经在该节点上运行的将不受影响。

```shell
 kubectl taint nodes 192.168.0.203 department=development:NoSchedule
```

对于一个节点上定义多个taint，必须所有的都满足后才能在节点上执行，否则不会被执行。下面的是定义一个NoExecute的taint，任何不能满足该条件的pod将会被立刻驱逐掉，在其他节点上被重新生成，

```
 kubectl taint nodes 192.168.0.203 department=development:NoExecute
```



在1.6版本后，加入了代表节点问题的taints，比如节点未准备好，内存或者磁盘空间不足，或者无法被调度等问题,1.13版本后这些taints会被自动增加到node上。

- node.kubernetes.io/out-of-disk
- node.kubernetes.io/memory-pressure
- node.kubernetes.io/network-unavailable
- ...

另外我们在配置pod的容忍上可以设置间隔，比如下面的pod设置最大容忍6000秒，即便是无法满足当前的网络不可达的taints:

```
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

#### 2.6 Job批量任务调度

Job调度是kubernetes针对批处理操作任务所设计的调度模式，这种模式下可以启动多个Job容器去执行任务处理，这里Job调度的主要模式分为：

1.  一个Job处理一项任务，job执行结束后退出，适合只有少数任务的比如，批量的日志处理操作
2. Job消费队列中的任务，启动固定的Pod执行任务
3. Job消费队列中的任务，但是启动Pod数量可变

比如下面的为一个Job的实例：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4

```

使用队列模式下的job可以利用redis存储任务，并定义自己的worker容器去消费redis中队列中的内容， 比如下面的配置将定义一个并行执行数量为2个的job，启动两个Pod来运行消费, 具体的实例介绍可以[参考官网Redis-Work-Queue](https://kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/)的介绍。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-2
spec:
  parallelism: 2
  template:
    metadata:
      name: job-wq-2
    spec:
      containers:
      - name: c
        image: gcr.io/myproject/job-wq-2
      restartPolicy: OnFailure
```



#### 2.5 CronJob定时任务调度

CronJob类似于Linux上运行的Cron任务，且定义的语法相似，定义一个周期执行的job，并设置每分钟执行一次，也就是每分钟生成一个pod执行后销毁

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```



```
$ kubectl get cronjob
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        38s             42s
```

执行下面的命令删除一个cron任务

```
kubectl delete cronjob hello
```

#### 2.7 DaemonSet后台运行调度

这种调度方式，允许在每一个node上执行一个指定的Pod，比如下面的是在每个Node上面启动一个日志处理任务，其中将主机的文件路径挂载到容器内部，并设置为只读。同时设置容器的资源限制，定义cpu和内存的使用。

定义tolerations来设置不受```node-role.kubernetes.io/master```限制的执行在任何主机上

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers	
```

在配置中如果定义了nodeSelector则只会在指定的节点上执行，如果定义了亲和度的话，也会根据亲和度的配置设置符合要求的node上执行，否则会在所有的节点上执行。

相对于直接在主机上配置系统启动或者静态的Pod， 这种方式允许我们监控Pod的执行状态。