---
layout: post
title: "Docker 1.12.0将要发布的新功能"
date: 2016-06-17 18:27:03 +0800
comments: true
categories: 
---


按计划，6/14 是1.12.0版本的 [feature冻结](https://github.com/docker/docker/wiki) 的日子，再有两个星期Docker 1.12.0也该发布了。这里列出来的新功能，都是已经合并到主分支的功能，不出意外，下一个版本的Docker应该是能体验到了。

下周2016 DockerCon也该开始了，好像也有一场专门来讲Docker新特性的，不过在这之前，我们就可以抢先一步，浏览一下这些新功能、新特性。尤其是前两个，都是比较吸引人的功能。


## Swarmkit集成

前几天Docker刚刚发布了 [Swarmkit](https://github.com/docker/swarmkit) ，也就是Swarm V2。

同时，在这个版本的Docker中，Swarm/Swarmkit 相关命令也被整合到了Docker子命令中。这可能算得上是1.12.0版本中最大的变更点了。

[这个PR（Add dependency to docker/swarmkit）](https://github.com/docker/docker/pull/23361) 有600个文件变动。
除了传统的image和container对象，这个PR增加了task和service等资源类型。

相关几个子PR包括 [23362](https://github.com/docker/docker/pull/23362) 、[23363](https://github.com/docker/docker/pull/23363) 、 [23364](https://github.com/docker/docker/pull/23364)。

使用新的Docker命令，可以这样直接创建Swarm集群：

```
$ docker swarm init --listen-addr 192.168.99.100:2377
Swarm initialized: current node (09fm6su6c24qn) is now a manager.
```


查看节点：

```
$ docker node ls
ID              NAME      MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS  LEADER
09fm6su6c24q *  manager1  Accepted    Ready   Active        Reachable       Yes
```

然后这样来部署一个新的service：

```
$ docker service create --scale 1 --name helloworld alpine ping docker.com
2zs4helqu64f3k3iuwywbk49w

$ docker service ls
ID            NAME        SCALE  IMAGE   COMMAND
2zs4helqu64f  helloworld  1      alpine  ping docker.com
```

需要scale了？没关系，也可以在Docker中直接完成：

```
$ docker service update --scale 5 helloworld
helloworld

$ docker service tasks helloworld
ID                         NAME          SERVICE     IMAGE   DESIRED STATE  LAST STATE          NODE
1n6wif51j0w840udalgw6hphg  helloworld.1  helloworld  alpine  RUNNING        RUNNING 2 minutes   manager1
dfhsosk00wxfb7j0cazp3fmhy  helloworld.2  helloworld  alpine  RUNNING        RUNNING 15 seconds  worker2
6cbedbeywo076zn54fnwc667a  helloworld.3  helloworld  alpine  RUNNING        RUNNING 15 seconds  worker1
7w80cafrry7asls96lm2tmwkz  helloworld.4  helloworld  alpine  RUNNING        RUNNING 10 seconds  worker1
bn67kh76crn6du22ve2enqg5j  helloworld.5  helloworld  alpine  RUNNING        RUNNING 10 seconds  manager1
```

在一台机器上使用 `docker ps` :

```
$ docker ps

CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
910669d5e188        alpine:latest       "ping docker.com"   10 seconds ago      Up 10 seconds                           helloworld.5.bn67kh76crn6du22ve2enqg5j
a0b6c02868ca        alpine:latest       "ping docker.com"   2 minutes  ago      Up 2 minutes                            helloworld.1.1n6wif51j0w840udalgw6hphg

```

我们也可以这样使用 `dcoekr service` 子命令。

```
$ docker service create --scale 3 --name redis --update-delay 10s --update-parallelism 1 redis:3.0.6

69uh57k8o03jtqj9uvmteodbb
$ docker service tasks redis
ID                         NAME     SERVICE  IMAGE        LAST STATE          DESIRED STATE  NODE
3wfqsgxecktpwoyj2zjcrcn4r  redis.1  redis    redis:3.0.6  RUNNING 13 minutes  RUNNING        worker2
8lcm041z3v80w0gdkczbot0gg  redis.2  redis    redis:3.0.6  RUNNING 13 minutes  RUNNING        worker1
d48skceeph9lkz4nbttig1z4a  redis.3  redis    redis:3.0.6  RUNNING 12 minutes  RUNNING        manager1
```


总之，新的Docker和Swarm结合在一起，管理集群和服务将会更方便。

## 插件管理（Plugin repository，experimental版）

PR地址： https://github.com/docker/docker/pull/23446

插件管理功能可能算是第二大变更点了。很多软件都支持插件机制，大家比较熟悉的从Wordpress到ElasticSearch等，都支持在软件内部通过plugin功能安装、管理插件。

Docker最近也增加了一些网络和卷管理的插件功能，这次还在体验版中增加了插件（基于容器）管理功能，可以通过 `docker plugin` 命令来管理插件。除了提供了一个统一的插件管理入口，还可以对插件的生命周期进行更好的管理，Docker君，我的插件写的比较不专业，请罩着我点。


这是一个大概的使用示意，通过 `docker plugin install` 可以安装插件， `docker plugin ls` 可以列出当前安装的插件：

```
$ docker plugin install aragunathan/no-remove
Plugin "aragunathan/no-remove:latest" requested the following privileges:
 - Networking: host
 - Mounting host path: /data
Do you grant the above permissions? [y/N] y
```

查看插件列表：

```
$ docker plugin ls
NAME                    VERSION             ACTIVE
aragunathan/no-remove   latest              true
```

目前 `docker plugin` 支持如下子命令：

* plugin ls
* plugin enable
* plugin inspect
* plugin install
* plugin rm


## 增加 `overlay2` 存储驱动（PR#22126 Overlay multiple lower directory support）

[这个PR](https://github.com/docker/docker/pull/22126) 增加一个新的名为 `overlay2` 的驱动，以解决Docker在存储优化方面的不足，充分利用4.0内核的 `lower directories` 新特性，解决inode耗尽等问题。

这个PR中的描述也提到了新旧overlay驱动的性能对比数据，有兴趣的可以参考一下。

## Live restore

[这个PR](https://github.com/docker/docker/pull/23213) 最大的好处就是提供了对daemonless容器的支持，即使daemon宕了，容器也不会受影响。

使用方法就是在启动 `dockerd` 的时候，增加 `--live-restore` 标志。

你是不是特别喜欢这个功能？

## Healthcheck

`docker run` 和 `Dockerfile` 新增加的 [健康检查功能](https://github.com/docker/docker/pull/23218)。

比如在 `docker run` 中，可以这样：

```
$ docker run --name=test -d \
    --health-cmd='stat /etc/passwd || exit 1' \
    --health-interval=2s \
    busybox sleep 1d
```

查看健康状态，返回 `healthy`：

```
$ sleep 2; docker inspect --format='{{.State.Health.Status}}' test
healthy
```

故意删除 `/etc/passwd` 文件：

```
$ docker exec test rm /etc/passwd
```

再次查看节点健康状态：

```
$ sleep 2; docker inspect --format='{{json .State.Health}}' test
{
  "Status": "unhealthy",
  "FailingStreak": 3,
  "Log": [
    {
      "Start": "2016-05-25T17:22:04.635478668Z",
      "End": "2016-05-25T17:22:04.7272552Z",
      "ExitCode": 0,
      "Output": "  File: /etc/passwd\n  Size: 334       \tBlocks: 8          IO Block: 4096   regular file\nDevice: 32h/50d\tInode: 12          Links: 1\nAccess: (0664/-rw-rw-r--)  Uid: (    0/    root)   Gid: (    0/    root)\nAccess: 2015-12-05 22:05:32.000000000\nModify: 2015..."
    },
    {
      "Start": "2016-05-25T17:22:06.732900633Z",
      "End": "2016-05-25T17:22:06.822168935Z",
      "ExitCode": 0,
      "Output": "  File: /etc/passwd\n  Size: 334       \tBlocks: 8          IO Block: 4096   regular file\nDevice: 32h/50d\tInode: 12          Links: 1\nAccess: (0664/-rw-rw-r--)  Uid: (    0/    root)   Gid: (    0/    root)\nAccess: 2015-12-05 22:05:32.000000000\nModify: 2015..."
    },
    {
      "Start": "2016-05-25T17:22:08.823956535Z",
      "End": "2016-05-25T17:22:08.897359124Z",
      "ExitCode": 1,
      "Output": "stat: can't stat '/etc/passwd': No such file or directory\n"
    }
  ]
}

```

## 其他改进

除了上述能单独拿出来的，要么比较重量级，要么非常实用的功能，Docker还有很多其他方面的改进。这里我们简单看看其中的一些。


### `dockerd` 和 `docker` 二进制文件分离

`docker daemon` 改为了 `dockerd` ，这样以后就需要使用两个可执行程序了。

PR： https://github.com/docker/docker/pull/22386


### 为 `btrfs` 存储驱动增加容器 `rootfs` 的限额功能

类似这样，可以在Docker守护进程或者容器级别设置根文件系统的大小：

```
$ docker daemon --storage-opt btrfs.min_space=xx
$ docker run --storage-opt size=xx
```

PR1：https://github.com/docker/docker/pull/19651
PR2：https://github.com/docker/docker/pull/19367

### `docker search` 增加 `--limit` 和 `filter` 参数

* `--limit` 参数

`docker search` 命令增加了一个 `--limit` 参数，可以设置查找镜像的返回结果个数。这个参数的默认值为25，可以接受的范围是0-100（实际上好像服务端最多只能返回100条记录）。

PR：https://github.com/docker/docker/pull/23107

* `--filter` 参数

比如可以指定只返回官方镜像，或者返回一定数量star的镜像：

```
$ docker search --filter is-official=true ubuntu

$ docker search --filter stars=30 ubuntu

```

PR：https://github.com/docker/docker/pull/22369

### `docker ps` 命令增加 `--filter network=xxx`参数

只列出指定网络模式的容器：

```
$ docker run -d --net=bridge --name=onbridgenetwork busybox top
$ docker run -d --net=none --name=onnonenetwork busybox top

$ docker ps --filter network=bridge 
```
PR： https://github.com/docker/docker/pull/23300

### `pid` 支持指定其他容器的pid

在之前的版本中，`pid` 参数只支持 `host` 模式，现在也可以和其他容器共享PID命名空间了。这也可能是一个比较实用的功能。

```
$ docker run --name my-redis -d redis

$ # 在其他容器中使用strace进行调试

$ docker run --it --pid=container:my-redis bash
$ strace -p 1
```

PR： https://github.com/docker/docker/pull/22481

### `docker network ls` 新增了两种过滤类型

`docker network ls` 也增加了两种类型的过滤

* 按驱动类型过滤

```
$ docker network ls --filter driver=bridge
NETWORK ID          NAME                DRIVER
db9db329f835        test1               bridge
f6e212da9dfd        test2               bridge
```

PR： https://github.com/docker/docker/pull/22319

* 按label过滤

```
$ docker network ls -f "label=usage"
NETWORK ID          NAME                DRIVER
db9db329f835        test1               bridge              
f6e212da9dfd        test2               bridge
```

PR: https://github.com/docker/docker/pull/21495

### CLI子命令重构

很多Docker 子命令都采用 [cobra](https://github.com/spf13/cobra) 对CLI进行了重构，应该不算是什么大的改动。


## 总结

当然，还有类似小的一些改进，这里并没有都列出来，到时候大家看一下参考手册就能见到了，也许很多功能大家根本就不会用到：-）

到这里我们简单的就浏览了一下新版本的Docker中将会搭载的新功能，如果你觉得哪些是你想要的，就等着Docker 1.12.0的发布吧。



