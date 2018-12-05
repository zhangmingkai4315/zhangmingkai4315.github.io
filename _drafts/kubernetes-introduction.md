## Kubernetes 入门

Kubernetes将多个服务器或者虚拟设备组成一个集群，协调集群内部的资源使用，容器调度和管理。自动分发和管理应用的容器,　所以要求**部署应用必须容器化**，相对于传统的应用直接部署在服务器上的方式，更加的灵活。

Kubernetes的两个角色：

- Master负责协调集群:
  - 调度应用
  - 管理应用状态
  - 应用的扩容
  - 升级应用等

- Nodes负责运行应用容器：可以是虚拟机或者一台实体服务器
  - 每一个Node包含一个kubelet，作为agent和master的通信并管理该节点
  - 相应的容器管理工具：docker或者rkt
  - 生产环境下需由至少三个nodes

用户使用KubernetesAPI可以直接和主进行通信进行容器管理



kubectl使用api和集群进行通信，　ubuntu安装kubectl的方式：

```sh

sudo apt-get update && sudo apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

CentOS的安装方式：

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl
```



查看集群中的节点：

```
kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   7m    v1.10.0
```

部署一个应用

```
kubectl run kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1 --port=8080
```

部署过程中会查找合适的节点去运行容器，调度容器在该节点上运行，配置集群当该节点不可用的时候去重新调度容器。

```sh
➜  ~ kubectl get deployments
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1         1         1            1           2m

```

Pods指的是kubernetes抽象的概念，代表了一个或者多个容器，以及容器之间共享的资源（存储，网络配置等）。一个pods模型是一个逻辑的应用主机，可以包含不同的紧密耦合在一起的应用容器，比如一个nodejs应用和后台的数据库容器以及缓存容器等。这些容器共享一个ip地址和端口空间，一般会在一个地方部署和调用。



Kubernetes创建的时候不是直接的创建容器，而是创建pods（这也是Kubernetes平台的原子单元），每一个pod和节点紧密绑定，除非节点失效后，相同的pods会在其他节点上被部署。

![](https://d33wubrfki0l68.cloudfront.net/fe03f68d8ede9815184852ca2a4fd30325e5d15a/98064/docs/tutorials/kubernetes-basics/public/images/module_03_pods.svg)



一个节点可以有多个pods，master在调度的时候将考虑nodes的可用资源进行调度分配。每一个kubernetes主机运行至少：

- kubelet，　进程用来master和node进行通信，管理运行在主机上的pods和容器
- 容器运行时（docker或者rkt）,负责容器管理，解压容器和运行应用。



![](https://d33wubrfki0l68.cloudfront.net/5cb72d407cbe2755e581b6de757e0d81760d5b86/a9df9/docs/tutorials/kubernetes-basics/public/images/module_03_nodes.svg)







运行在一个私有的隔离的网络中，集群中其他的Pods和服务可以相互通信，但是无法被外界访问。

Pods的名字可以通过下面的命令查询:

```
POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

一些常用的操作：

```sh
➜  ~ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-5c69669756-26jqh   1/1     Running   0          28m

➜  ~ kubectl describe pods
Name:           kubernetes-bootcamp-5c69669756-26jqh
Namespace:      default
Node:           minikube/10.0.2.15
Start Time:     Wed, 05 Dec 2018 20:54:55 +0800
Labels:         pod-template-hash=1725225312
                run=kubernetes-bootcamp
Annotations:    <none>
Status:         Running
IP:             172.17.0.5
Controlled By:  ReplicaSet/kubernetes-bootcamp-5c69669756
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://63ccca7af295a8ceb43227258be7973baffa3481ef81b804c5efe1b1718c02a7
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v1
    Image ID:       docker-pullable://gcr.io/google-samples/kubernetes-bootcamp@sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 05 Dec 2018 20:55:54 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-kfrqv (ro)
Conditions:
  Type           Status
  Initialized    True 
  Ready          True 
  PodScheduled   True 
Volumes:
  default-token-kfrqv:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-kfrqv
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason                 Age   From               Message
  ----    ------                 ----  ----               -------
  Normal  Scheduled              28m   default-scheduler  Successfully assigned kubernetes-bootcamp-5c69669756-26jqh to minikube
  Normal  SuccessfulMountVolume  28m   kubelet, minikube  MountVolume.SetUp succeeded for volume "default-token-kfrqv"
  Normal  Pulling                28m   kubelet, minikube  pulling image "gcr.io/google-samples/kubernetes-bootcamp:v1"
  Normal  Pulled                 28m   kubelet, minikube  Successfully pulled image "gcr.io/google-samples/kubernetes-bootcamp:v1"
  Normal  Created                28m   kubelet, minikube  Created container
  Normal  Started                28m   kubelet, minikube  Started container
  
  
➜  ~ kubectl logs kubernetes-bootcamp-5c69669756-26jqh
Kubernetes Bootcamp App Started At: 2018-12-05T12:55:54.081Z | Running On:  kubernetes-bootcamp-5c69669756-26jqh 

Running On: kubernetes-bootcamp-5c69669756-26jqh | Total Requests: 1 | App Uptime: 460.297 seconds | Log Tim

kubectl exec -it kubernetes-bootcamp-5c69669756-26jqh bash
root@kubernetes-bootcamp-5c69669756-26jqh:/# 

```



当一个节点失效的时候，在节点上运行的Ｐods也将会丢失，但复制控制器动态的创建新的pods去提供服务。假如后端服务重新启动后端口发生变化，前端系统不应该去关注任何后端服务端口的变化。每一个pod都有一个独立的ip地址，需要有一种方式在pods之间进行自动调整，保证应用的运行。



服务service是另一个抽象的内容，定义一组pods的逻辑和访问策略。解耦pods直接的通信。一个服务可以使用yaml文件进行定义(或者json).

尽管一个pod都有自己独特的ip地址，但是这些ip不会暴露在外面，使用服务允许应用接收外面的流量,主要方式：

- 集群IP, 服务暴露一个集群内部使用的ip，只能在内部使用
- node端口，服务暴露在选择的主机的端口上，`<NodeIP>:<NodePort>`的形式被外界访问
- 负载均衡：　创建一个外部负载均衡，分配一个固定的外部IP地址
- 外部名称，使用一个域名来暴露服务，返回一个CNAME记录



![](https://d33wubrfki0l68.cloudfront.net/cc38b0f3c0fd94e66495e3a4198f2096cdecd3d5/ace10/docs/tutorials/kubernetes-basics/public/images/module_04_services.svg)

kubernetes的服务可以帮助路由不同pods的流量，管理相互之间的通信, 通过标签选择器来选择不同的pods



![](https://d33wubrfki0l68.cloudfront.net/b964c59cdc1979dd4e1904c25f43745564ef6bee/f3351/docs/tutorials/kubernetes-basics/public/images/module_04_labels.svg)

```
➜  ~ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-5c69669756-26jqh   1/1     Running   0          47m
➜  ~ 

查看服务列表
➜  ~ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   56m
➜  ~ 

创建一个服务
➜  ~ kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
service/kubernetes-bootcamp exposed
➜  ~ 
➜  ~ kubectl get services                                                       
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes            ClusterIP   10.96.0.1       <none>        443/TCP          57m
kubernetes-bootcamp   NodePort    10.108.131.67   <none>        8080:32635/TCP   3s
➜  ~ 
➜  ~ kubectl describe services/kubernetes-bootcamp
Name:                     kubernetes-bootcamp
Namespace:                default
Labels:                   run=kubernetes-bootcamp
Annotations:              <none>
Selector:                 run=kubernetes-bootcamp
Type:                     NodePort
IP:                       10.108.131.67
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  32635/TCP
Endpoints:                172.17.0.5:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
➜  ~ minikube ip
192.168.99.100
```

