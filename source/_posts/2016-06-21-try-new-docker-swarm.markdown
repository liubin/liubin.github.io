---
layout: post
title: "初试Docker swarm"
date: 2016-06-21 20:41:42 +0800
comments: true
categories: 
---

DockerCon 2016今天凌晨开始了，Docker公司的CEO和CTO都做了慷慨激昂的演讲（其实是我瞎想象的，没听，只是看了一些别人整理的 highlight ）。

同时，1.12 RC2也出来了，很多 [新功能](http://liubin.org/blog/2016/06/17/whats-new-in-docker-1-dot-12-dot-0/) 已经可以试用了， 这里我们就来试一下 `docker swarm` 命令吧，这也是 `swarm` 成为Docker子命令的第一个版本。


测试环境：

- OS：OS X
- Docker：1.12.0-rc2
- Docker Machine：0.8.0-rc1


系统构成：

- master 1台
- worker 2台

## 升级Docker

使用Docker for Mac升级Docker很简单，这里不多说了；升级之后确认一下是最新版的Docker就可以了。

## 创建3台Docker主机

首先，使用Docker Machine创建3台主机，一台名为 `manager1` ，其余两台为 `worker1` 和 `worker2` ：


创建master节点：
```
$ docker-machine create --driver virtualbox manager1
```

创建2个worker节点：

```
$ docker-machine create --driver virtualbox worker1
$ docker-machine create --driver virtualbox worker2
```

之后可以确认一下3台机器是否创建成功，以及ta们的IP地址：

```
$ docker-machine ls
NAME       ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
manager1   -        virtualbox   Running   tcp://192.168.99.100:2376           v1.12.0-rc2   
worker1    -        virtualbox   Running   tcp://192.168.99.101:2376           v1.12.0-rc2   
worker2    -        virtualbox   Running   tcp://192.168.99.102:2376           v1.12.0-rc2   

```

## 创建Swarm集群


首先，切换到 `manager1` 主机，使用 `docker swarm init` 命令创建一个集群：

```
$ eval $(docker-machine env manager1)

$ docker swarm init --listen-addr 192.168.99.100:2377
Swarm initialized: current node (alq8w7fi34f41j3z4ise1vkd7) is now a manager.
```

使用 `docker info` 确认一下当前节点信息，可以看到Swarm属性部分：

```
$ docker info
... ...
Swarm: active
 NodeID: alq8w7fi34f41j3z4ise1vkd7
 IsManager: Yes
 Managers: 1
 Nodes: 1
 CACertHash: sha256:7a9d0eb1621afe2be07c5fd405b8f038c76be3a8dc7b2c73944a2d1ab9dffd76

... ...
```

在2台worker节点上，通过  `docker swarm join` 命令加入到刚才创建的集群中：

```
$ eval $(docker-machine env worker1)
$ docker swarm join 192.168.99.100:2377
This node joined a Swarm as a worker.
$ docker info
... ...
Swarm: active
 NodeID: 4l2a9ebgmcpwqlo0roye0n6m5
 IsManager: No
... ...
```

回到 `manager1` 节点，确认一下集群中节点的个数和状态。在 `MANAGER STATUS` 属性中，可以看到谁是Leader：

```
$ eval $(docker-machine env manager1)
$ docker node ls
ID                           NAME      MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
4l2a9ebgmcpwqlo0roye0n6m5    worker1   Accepted    Ready   Active        
alq8w7fi34f41j3z4ise1vkd7 *  manager1  Accepted    Ready   Active        Leader
e0khh79c6owm0e14mli602q0a    worker2   Accepted    Ready   Active        
```

## 创建nginx服务

首先，创建一个覆盖网络：

```
$ docker network create -d overlay ngx_net
8kiv8muduf60f66rs99ufo25f
```

然后使用这个覆盖网络，创建nginx服务：

```
$ docker service create --name nginx --replicas 1 --network ngx_net -p 80:80/tcp nginx
78cmmh8ef4qcwmjjzgn3k45ch
```

这样，就创建了一个具有一个副本（ `--replicas 1` ）的 `nginx` 服务，使用镜像 `nginx` 。

```
$ docker service tasks nginx
ID                         NAME     SERVICE  IMAGE  LAST STATE           DESIRED STATE  NODE
394by1b72i44a44jms2xwk6ud  nginx.1  nginx    nginx  Preparing 9 seconds  Running        manager1
```


注意上面的 `STATE` 字段中刚开始的服务状态为 `Preparing`，需要等一会才能变为 `Running` 状态，其中最费时间的应该是下载镜像的过程。

过一会再查看服务状态，就可以看到状态已经变为 `Running` 了，这是可以通过 `http://192.168.99.100/` 查看 Nginx服务。

```
$ docker service tasks nginx
ID                         NAME     SERVICE  IMAGE  LAST STATE              DESIRED STATE  NODE
394by1b72i44a44jms2xwk6ud  nginx.1  nginx    nginx  Running About a minute  Running        manager1

# 通过curl查看服务是否正常运行
$ curl http://192.168.99.100/
...

```

## 对服务进行扩展（scale）


当然，如果只是通过service启动容器，swarm也算不上什么新鲜东西了。Service还提供了复制（类似k8s里的副本）功能。可以通过 `docker service scale` 命令来设置服务中容器的副本数：

```
$ docker service scale nginx=5
nginx scaled to 5
```

和创建服务一样，增加scale数之后，将会创建新的容器，这些新启动的容器也会经历从准备到运行的过程，过一分钟左右，服务应该就会启动完成，这时候可以再来看一下 `nginx` 服务中的容器（task）：

```
$ docker service tasks nginx
ID                         NAME     SERVICE  IMAGE  LAST STATE         DESIRED STATE  NODE
394by1b72i44a44jms2xwk6ud  nginx.1  nginx    nginx  Running 5 minutes  Running        manager1
co0re9u7infoo9qiegm6yiqcn  nginx.2  nginx    nginx  Running 2 minutes  Running        worker1
19dvayah8fjz3vykrl2oi12uu  nginx.3  nginx    nginx  Running 2 minutes  Running        worker1
d8okdip767972p083tix4dk7d  nginx.4  nginx    nginx  Running 2 minutes  Running        manager1
9rq59mf6bq5m411y6gdzb5pq6  nginx.5  nginx    nginx  Running 2 minutes  Running        worker2
```

可以看到，之前 `nginx` 容器只在 `manager1` 上有一个实例，而现在又增加了4个实例。

我们可以在 `manager1` 上查看一下这台主机上运行的容器：

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
4bbc02bc426e        nginx:latest        "nginx -g 'daemon off"   4 minutes ago       Up 4 minutes        80/tcp, 443/tcp     nginx.4.d8okdip767972p083tix4dk7d
1eda6f7d3029        nginx:latest        "nginx -g 'daemon off"   5 minutes ago       Up 5 minutes        80/tcp, 443/tcp     nginx.1.394by1b72i44a44jms2xwk6ud
```


## 容器异常停止时的处理

如果一个服务中的一个任务突然终止了，Docker会怎么处理？这里我们就来模拟某一容器异常终止的情况。

在操作之前，我们在另一个窗口，准备好使用 `tail -f /var/log/docker.log` 命令来查看Docker守护进程的日志。

我们通过 `docker rm -f` 来删除一个容器（`4b` 表示的是我们第一个启动的 `4bbc02bc426e` 容器）：

```
$ docker rm -f 4b
4b

```

回到Docker守护进程的日志查看窗口，我们会看到类似这样的日志（有删减），从日志中，我们可以看到旧容器（id为 `4bbc02bc426e` ，task id为 `d8okdip767972p083tix4dk7d`）的删除和新容器（task id为 `cumjdktbadaxca66rt4hi63na`）的调度过程（删除了日志中的时间戳和日志级别等非重要信息，但保留了日志的时间顺序）：

```
# 调用删除容器的API
msg="Calling DELETE /v1.24/containers/4b?force=1" 

# d8的任务从RUNNING变为了FAILED状态
msg="state changed" module=taskmanager state.desired=RUNNING state.transition="RUNNING->FAILED" task.id=d8okdip767972p083tix4dk7d 

# 将旧任务停止
msg=assigned module=agent task.desiredstate=SHUTDOWN task.id=d8okdip767972p083tix4dk7d 

# 分配到新的节点，创建新的任务id： cumj
msg="Assigning to node e0khh79c6owm0e14mli602q0a" task.id=cumjdktbadaxca66rt4hi63na 

# 旧任务状态更新为FAILED
msg="(*Agent).UpdateTaskStatus" module=agent task.id=d8okdip767972p083tix4dk7d 
msg="task status updated" method="(*Dispatcher).processTaskUpdates" module=dispatcher state.transition="FAILED->FAILED" task.id=d8okdip767972p083tix4dk7d 

# 新任务从ASSIGNED变为接受状态
msg="task status updated" method="(*Dispatcher).processTaskUpdates" module=dispatcher state.transition="ASSIGNED->ACCEPTED" task.id=cumjdktbadaxca66rt4hi63na 

# 准备运行新任务
msg="task status updated" method="(*Dispatcher).processTaskUpdates" module=dispatcher state.transition="ACCEPTED->PREPARING" task.id=cumjdktbadaxca66rt4hi63na 

# 运行新任务
msg="task status updated" method="(*Dispatcher).processTaskUpdates" module=dispatcher state.transition="PREPARING->STARTING" task.id=cumjdktbadaxca66rt4hi63na 

# 新任务启动完成
msg="task status updated" method="(*Dispatcher).processTaskUpdates" module=dispatcher state.transition="STARTING->RUNNING" task.id=cumjdktbadaxca66rt4hi63na 

```

从上面的日志我们不难看出，一个任务的生命周期的前半生，大概就是 `ASSIGNED` -> `ACCEPTED` -> `PREPARING` -> `STARTING` -> `RUNNING` 。


在 `manager1`主机上，我们看到这个容器已经删除了， `ps` 只能看到一个容器：

```
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
1eda6f7d3029        nginx:latest        "nginx -g 'daemon off"   6 minutes ago       Up 6 minutes        80/tcp, 443/tcp     nginx.1.394by1b72i44a44jms2xwk6ud
```

再来查看一下 `nginx` 服务的任务列表，可以看到新创建的任务 `cumjdktbadaxca66rt4hi63na` 被调度到了 `worker2` 上运行：

```
$ docker service tasks nginx
ID                         NAME     SERVICE  IMAGE  LAST STATE          DESIRED STATE  NODE
394by1b72i44a44jms2xwk6ud  nginx.1  nginx    nginx  Running 8 minutes   Running        manager1
co0re9u7infoo9qiegm6yiqcn  nginx.2  nginx    nginx  Running 5 minutes   Running        worker1
19dvayah8fjz3vykrl2oi12uu  nginx.3  nginx    nginx  Running 5 minutes   Running        worker1
cumjdktbadaxca66rt4hi63na  nginx.4  nginx    nginx  Running 32 seconds  Running        worker2
9rq59mf6bq5m411y6gdzb5pq6  nginx.5  nginx    nginx  Running 5 minutes   Running        worker2
```


## 删除一个节点？

不难想象，如果一个节点都宕机了，则Docker应该会将在该节点运行的容器，调度到其他节点，以满足指定数量的副本保持运行状态。

下面我们就来模拟一下这种场景。

首先，我们删除一个节点 `worker2`：

```
$ docker-machine rm worker2
About to remove worker2
Are you sure? (y/n): y
Successfully removed worker2
```

删除之后，Docker就会开始重新调度，最终调度结束（< 1分钟）后，再查看该服务的任务状态，应该如下面这样，有5个 `nginx` 容器在剩下的两台机器上运行：

```
$ docker service tasks nginx
ID                         NAME     SERVICE  IMAGE  LAST STATE          DESIRED STATE  NODE
394by1b72i44a44jms2xwk6ud  nginx.1  nginx    nginx  Running 22 minutes  Running        manager1
co0re9u7infoo9qiegm6yiqcn  nginx.2  nginx    nginx  Running 19 minutes  Running        worker1
19dvayah8fjz3vykrl2oi12uu  nginx.3  nginx    nginx  Running 19 minutes  Running        worker1
991v97eg9q1hnnzxda6c9mmv7  nginx.4  nginx    nginx  Running 20 seconds  Running        manager1
e9yztfmy5luaxnadz80e5j8nl  nginx.5  nginx    nginx  Running 20 seconds  Running        manager1
```


除了上面用到的一些命令， `docker service` 还有以下一些子命令：


```
$ docker service --help

Usage:	docker service COMMAND

Manage Docker services

Options:
      --help   Print usage

Commands:
  create      Create a new service
  inspect     Inspect a service
  tasks       List the tasks of a service
  ls          List services
  rm          Remove a service
  scale       Scale one or multiple services
  update      Update a service

Run 'docker service COMMAND --help' for more information on a command.
```

比如我们可以用 `docker service inspect` 来获得 `nginx` 服务的详情信息：

```
$ docker service inspect nginx
[
    {
        "ID": "78cmmh8ef4qcwmjjzgn3k45ch",
        "Version": {
            "Index": 32
        },
        "CreatedAt": "2016-06-21T08:31:21.427244594Z",
        "UpdatedAt": "2016-06-21T08:33:50.75625288Z",
        "Spec": {
            "Name": "nginx",
            "TaskTemplate": {
                "ContainerSpec": {
                    "Image": "nginx"
                },
                "Resources": {
                    "Limits": {},
                    "Reservations": {}
                },
                "RestartPolicy": {
                    "Condition": "any",
                    "MaxAttempts": 0
                },
                "Placement": {}
            },
            "Mode": {
                "Replicated": {
                    "Replicas": 5
                }
            },
            "UpdateConfig": {},
            "Networks": [
                {
                    "Target": "8kiv8muduf60f66rs99ufo25f"
                }
            ],
            "EndpointSpec": {
                "Mode": "vip",
                "Ports": [
                    {
                        "Protocol": "tcp",
                        "TargetPort": 80,
                        "PublishedPort": 80
                    }
                ]
            }
        },
        "Endpoint": {
            "Spec": {},
            "Ports": [
                {
                    "Protocol": "tcp",
                    "TargetPort": 80,
                    "PublishedPort": 80
                }
            ],
            "VirtualIPs": [
                {
                    "NetworkID": "e9et32s47olva4e0uisamdqao",
                    "Addr": "10.255.0.6/16"
                },
                {
                    "NetworkID": "8kiv8muduf60f66rs99ufo25f",
                    "Addr": "10.0.0.2/24"
                }
            ]
        }
    }
]
```

也可以通过 `docker service rm nginx` 命令，删除 `nginx` 服务（现在的版本删除服务前没有警告提示，请小心操作）。

## 总结

上手很简单，Docker swarm可以非常方便的创建类似k8s那样带有副本的服务，确保一定数量的容器运行，保证服务的高可用。

然而，光从官方文档来说，功能似乎又有些简单，从生产环境来说，下面这些方面都还有所欠缺（其实从Swarm v1就有这个问题）：

- 没有配置文件，不好进行版本化管理，不方便部署
- 持久存储的缺失。Docker已经支持Volume Driver，k8s等也有存储卷插件在机制，真正的生产环境没有Volume支持估计是很难想象的。
- 是否能确保本身的高可用
- 调度策略还处于初级阶段

不过，正如Docker让容器技术变得平民化一样，Docker Machine和Swarm，也将在各种基础设施上运行Docker和Docker集群变得更加简单，从这一点上来说，其意义也是很大的。

不过在开源社区和商业竞争的角度来看，Docker Swarm将会走向何方呢？


