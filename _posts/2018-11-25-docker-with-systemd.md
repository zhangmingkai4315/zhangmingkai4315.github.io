---
layout: post
title: 使用Systemd管理单宿主下的Docker容器
date: 2018-11-25T11:32:41+00:00
author: mingkai
permalink: /2018/11/
categories:
  - docker
  - devops
---
Docker容器运行在宿主机上，为了保证容器承载服务的稳定，需要利用一些服务管理器来管理Docker容器的运行状态。Systemd作为一个系统守护进程，通过独立单元的形式管理系统上的服务，比如进程，脚本执行，挂载服务等等，同样可以直接管理容器的启动和关闭。

### 1. 管理单一容器

下面是一个启动管理配置文件，用于管理一个nginx容器的启动和删除。

```
[Unit]
Description=Simple Web Server
After=docker.service
Requires=docker.service

[Service]
Restart=always
ExecStartPre=/bin/bash -c '/usr/bin/docker rm -f web-service || /bin/true'
ExecStartPre=/usr/bin/docker pull nginx

ExecStart=/usr/bin/docker run --name web-service -p 80:80 nginx
ExecStop=/usr/bin/docker rm -f web-service

[Install]
WantedBy=multi-user.target

```

文件创建在```/etc/systemd/system```文件夹下，使用下面的命令进行管理即可：

```
  sudo systemctl enable /etc/systemd/system/web.service
  sudo systemctl start web.service
  sudo systemctl stop web.service

```

如果修改了配置的话需要重新执行载入命令来加载新的配置项目

```
  sudo systemctl daemon-reload
```



> 配置文件中的路径必须是绝对路径



### 2. 管理多个容器

通过docker-compose可以管理多个容器的运行和启动，同时可以利用systemd来管理docker-compose启动和管理. 以下是一个针对docker compose的配置文件```docker-compose.service```:

```
[Unit]
Description=Docker Compose container starter
After=docker.service network-online.target
Requires=docker.service network-online.target

[Service]
WorkingDirectory=/etc/compose
Type=oneshot
RemainAfterExit=yes

ExecStartPre=-/usr/local/bin/docker-compose pull --quiet
ExecStart=/usr/local/bin/docker-compose up -d

ExecStop=/usr/local/bin/docker-compose down

ExecReload=/usr/local/bin/docker-compose pull --quiet
ExecReload=/usr/local/bin/docker-compose up -d

[Install]
WantedBy=multi-user.target
```

这里配置的类型```Type=oneshot```和缺省的simple类型像是，但是期待执行的任务会在运行后退出，配合```RemainAfterExit=yes``` 定义在启动程序成功退出的时候依然认为任务是活跃的，缺省是no.

同时可以配合两个额外的脚本实现重载和定时任务.

docker-compose-reload.service

```
[Unit]
Description=Refresh images and update containers

[Service]
Type=oneshot

ExecStart=/bin/systemctl reload-or-restart docker-compose.service
```

docker-compose-reload.timer, 设置每隔15分钟执行一次重载，调用docker-compose-reload服务。

```
[Unit]
Description=Refresh images and update containers
Requires=docker-compose.service
After=docker-compose.service

[Timer]
OnCalendar=*:0/15

[Install]
WantedBy=timers.target
```



定时任务添加的时候可以使用类似于服务相同的命令添加：

```
# systemctl enable docker-compose-reload.timer
# systemctl start docker-compose-reload.timer
# systemctl is-active docker-compose-reload.timer
active
```

任何时候想要直接执行都可以直接调用下面的命令来运行一次重载

```
# systemctl start docker-compose-reload
```

检查当前系统中的所有已配置的timer

```
# systemctl list-timers
```

