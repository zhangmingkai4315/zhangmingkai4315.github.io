---
layout: post
title: Kubernetes共享存储配置和使用
date: 2019-01-17T11:32:41+00:00
author: mingkai
permalink: /2019/01/kubernetes-peristent-volume
categories:
  - kubernetes
  - docker
---

### Kubernetes共享存储

Kubernetes中为了持久保存数据，简化存储的分配和管理，利用PersitentVolume和PersistentVolumeClaim来管理存储资源并绑定到具体的Pod中使用。

- PersistentVolume是对底层网络存储的一种抽象，可以看作一个预先创建好的资源池。PV由管理员创建和配置，与底层的存储方式直接关联。定义后可以和PVC进行绑定
- PersistentVolumeClaim： 用户利用PVC去指定的PV(资源池中)申请存储资源。存储资源必须从PV中申请，PVC包含了申请的存储空间和访问的方式。
- StorageClass: 尽管用户可以直接使用PVC去申请资源，但是为了区分不同的存储资源的特性（SSD, RAM, 慢速存储，存储的供应商等等）管理员可以通过StorageClass来定义存储的类别，用户在申请存储时候可以根据StorageClass申请不同的存储资源，而且后面使用的动态分配的方式也是基于StorageClass来完成。

> 定义了StorageClass的PV只能被设置了对应Class的PVC请求， 没有设置类别的PV也只能与没有设置对应类别的PVC绑定



#### 1. 共享存储的管理

存储的生命周期管理主要涉及PV的创建，以及PVC与PV的后期交互。如下图所示：

![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1548402412&di=16cfe4c99f4a570ec039ebfb1c2e9ca8&imgtype=jpg&er=1&src=http%3A%2F%2Fpic4.zhimg.com%2Fv2-a8eed906c1dd6e3fb8129cfddf778a3b_b.png)

首先为了用户可以使用共享存储，需要创建存储的提供方式，主要分为动态和静态分配两种方式，

- 静态： 管理员预先创建一系列的PV，用户可以从这些创建好的PV中申请资源并使用
- 动态：当没有任何PV符合PVC的需求的时候，集群可以利用StorageClasses动态的创建资源（管理员需要预先设置好对应的Class类别和）

> 为了开启动态的存储申请，需要API服务器在启动的时候在--enable-admission-plugins中加入DefaultStorageClass



用户创建PVC后，提交申请，kubernetes将根据用户的PVC中的要求进行查询，是否有符合的PV满足需求，一旦找到则直接进行绑定，且绑定后不能与其他的PVC结合。对于查询后找不到符合要求的，对于指定了缺省的StorageClass的则自动创建一个PV并绑定，否则处于pending状态。

用户使用存储的方式则是通过创建Pod的时候，指定volume来使用，volume类型则为PersistentVolumeClaim, 多个Pod可以共享一个PVC（根据之前创建的访问策略决定）。当Pod被删除后，或者删除PVC， 则与该PVC绑定的PV会标记为已释放，但是还不能使用，除非进行资源的回收。管理员可以设置回收策略。

- Retain保留策略： 允许手动的回收资源，当PVC被删除的时候，PV仍会存在且标记为released已释放状态，但是此时无法被使用，因为还保留了原有的数据，管理员可以删除PV(外部的云存储环境下关联的存储仍会存在)， 手动的删掉关联的存储数据。
- Delete删除策略： 当删除PVC时候，自动的删除PV和实际的存储数据。动态分配的缺省为Delete模式，管理员应该根据实际的需求进行配置。或者如下面所示给缺省default=Delete的打补丁修改为Retain

```shell
kubectl patch pv <your-pv-name> -p '{"spec"{"persistentVolumeReclaimPolicy":"Retain"}}'
```

- Recycle回收策略：回收策略会简单的执行rm -rf来删除对应的volume使其可以被重新使用



不同的存储方式可能对于上述的支持情况不同，使用时候应该阅读相关的文档，确认使用方式。



#### 2  基于主机路径的共享存储

以下是一个PV和一个PVC的例子, 这里的PV方式是hostPath也就是直接利用node主机上的指定路径作为存储介质，访问的方式是ReadWriteOnce也就是只能被单一的Node使用，允许进行读写。其他的访问方式包括：ReadWriteMany（读写，多个Node挂载）和ReadOnlyMany(只读，多个Node挂载)

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
    
---

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

pvc可以直接通过storageClass申请到指定类型的存储并与其进行绑定，

```
❯ kubectl get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS   REASON   AGE
task-pv-volume   10Gi       RWO            Retain           Bound    default/task-pv-claim   manual                  3m23s
                                                                                                            
❯ kubectl get pvc
NAME            STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
task-pv-claim   Bound    task-pv-volume   10Gi       RWO            manual         21s

```

pvc申请后还需要绑定到指定的Pod上才能正常使用，下面的配置将创建一个Pod并绑定之前已经创建好的Volume,  通过观察发现，当我们创建pv,和pvc时候，并没有在任何的Node节点上创建指定的文件， 只有当我们创建pod的时候才真正的出现指定的路径。

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
       claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

使用hostPath的方式，存在的问题是，除非我们指定Pod在指定的Node上执行，否则当Pod重启或者在其他Node上被启动后，将找不到指定的文件，因此为了满足需求，一般利用网络存储NFS或者CephFS等网络存储的方式。

#### 3  基于网络存储的实例

这里以GlusterFS为实例介绍如何配置网络存储，首先在需要组成集群的机器上创建对应的块存储，并设置自动启动绑定，以下为Ansible的配置任务文件:

- 创建文件夹
- 创建磁盘块大小为100GB
- 绑定磁盘到设备
- 利用systemd设置服务启动自动绑定设备和磁盘
- 在每台机器上安装glusterfs-client （一定要确保安装，否则后面执行报错）

```yaml
- name: create gluster file folder in every node
  file:
    path: /home/glusterfs
    state: directory
  when: recreate == "yes"
- name: create glusterimages for 100GB in each node
  command: "dd if=/dev/zero of=/home/glusterfs/glusterimage bs=1M count=100000"
  when: recreate == "yes"

- name: create loop device to bind images
  command: "losetup /dev/loop100 /home/glusterfs/glusterimage"
  when: recreate == "yes"

- name: copy systemd config file for bootup device bind operation
  copy:
    src: "loopback_gluster.service"
    dest: "/etc/systemd/system/loopback_gluster.service"

- name: enable config for loopback file system on boot
  command: "systemctl enable /etc/systemd/system/loopback_gluster.service "
  
- name: install centos-release-gluster
  yum:
    name: centos-release-gluster
    state: installed

- name: install glusterfs-client
  yum: 
    name: glusterfs-client
    state: installed
```

需要复制到集群中的配置文件loopback_gluster.service如下所示：

```yaml
[Unit]
Description=Create the loopback device for glusterfs
DefaultDependencies=false
Before=local-fs.target
After=systemd-udev-settle.service
Requires=systemd-udev-settle.service

[Service]
Type=oneshot
ExecStart=/usr/bin/bash -c "losetup /dev/loop100 /home/glusterfs/glusterimage"

[Install]
WantedBy=local-fs.target
```

下载gluster的官方kubernetes安装脚本： https://github.com/gluster/gluster-kubernetes.git， 下载后进入deploy文件夹下修改配置信息

```sh
cd deploy
mv topology.json.sample topology.json
```

根据实际情况进行修改配置文件topology.json, 主要是修改nodes中manage, storage和devices的配置，其中manage和storage配置实际的IP地址，devices配置之前生成的磁盘

```
"nodes": [{
          "node": {
            "hostnames": {
              "manage": [
                "192.168.10.102"
              ],
              "storage": [
                "192.168.10.102"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/loop100"
          ]
  }
  ...
```

配置完成后执行如下的命令

```
./gk-deploy -g
```

如果中间执行过程中出现问题中断，重复执行可能导致存储的初始化失败， 最简单的方式是重新删除原有的磁盘，重新配置topo文件执行安装部署。

部署一切正常后，会输出一些辅助信息，比如提示如何创建StorageClass， 我的脚本输出信息包含的配置如下所示：

```yaml
---
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.44.0.3:8080"
```

使用kubectl创建该资源后即可使用该存储类，注意GlusterFS默认设置的为3副本，因此为了不丢失数据，最少的主机数量为3台。下面我们使用一个MySQL1主2从的例子来说明如何使用Glusterfs存储作为后端存储， 通过使用kubernertes的

创建MySQL的配置文件：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels: 
    app: mysql
data:
  master.cnf: |
    # master config
    [mysqld]
    log-bin
  slave.cnf: |
    # slave config
    [mysqld]
    super-read-only

```

创建MySQL服务文件, 这里使用mysql-0.mysql服务作为主MySQL服务进行写入，mysql-read作为读取数据的服务入口,前提是集群的DNS解析服务正常的情况下。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql

---

apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql

```

最后一部分就是实际进行Stateful应用的部署部分，内容比较多包含了initContainers和containers两个部分，其中initContainers包含了配置文件的初始化阶段，利用stateful本身的一些特性比如pod标签来提取其中的一些启动信息到内部的配置文件中，第一个创建的将作为mysql的master，后面的则作为slave来进行配置。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: mysql
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Generate mysql server-id from pod ordinal index.
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # Add an offset to avoid reserved server-id=0 value.
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # Copy appropriate conf.d files from config-map to emptyDir.
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      - name: clone-mysql
        image: zhangmingkai4315/google-samples-xtrabackup:latest
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Skip the clone if data already exists.
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Skip the clone on master (ordinal index 0).
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # Clone data from previous peer.
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # Prepare the backup.
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # Check we can execute queries over TCP (skip-networking is off).
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      - name: xtrabackup
        image: zhangmingkai4315/google-samples-xtrabackup:latest
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql

          # Determine binlog position of cloned data, if any.
          if [[ -f xtrabackup_slave_info ]]; then
            # XtraBackup already generated a partial "CHANGE MASTER TO" query
            # because we're cloning from an existing slave.
            mv xtrabackup_slave_info change_master_to.sql.in
            # Ignore xtrabackup_binlog_info in this case (it's useless).
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # We're cloning directly from master. Parse binlog position.
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi

          # Check if we need to complete a clone by starting replication.
          if [[ -f change_master_to.sql.in ]]; then
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done

            echo "Initializing replication from clone position"
            # In case of container restart, attempt this at-most-once.
            mv change_master_to.sql.in change_master_to.sql.orig
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi

          # Start a server to send backups when requested by peers.
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
  volumeClaimTemplates:
  - metadata:
      name: data
      annotations:
        volume.beta.kubernetes.io/storage-class: glusterfs-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi

```

创建完成后，添加一些数据到主数据库，既可以看到slave的自动复制和数据同步。

