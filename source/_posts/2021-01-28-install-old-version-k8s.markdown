---
layout: post
title: "安装老版本的 Kubernetes"
date: 2021-01-28 14:56:13 +0800
comments: true
categories: 
---

记得前些日子看过一个报告，说是私有部署中公司内部用的 Kubernetes ，要比社区版延迟有 17 个月。所以，在很多公司，安装一个老版本的 Kubernetes ，可能还真是一个日常操作。

不过不光 Kubernetes 有版本问题，连它的安装工具 kubeadm 也是有版本要求的，要想安装指定版本的 Kubernetes ，还得安装指定版本的 kubeadm 。

这里以在 Ubuntu 下为例，记录一下如何安装一个 1.16 版本的 Kubernetes。

可以使用 `apt-cache` 命令查看支持的 Kubernetes 版本。

```
# apt-cache madison kubectl | grep 1.16
   kubectl | 1.16.15-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl | 1.16.14-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl | 1.16.13-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl | 1.16.12-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl | 1.16.11-01 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl | 1.16.11-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl | 1.16.10-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.9-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.8-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.7-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.6-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.5-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.4-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.3-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.2-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.1-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubectl |  1.16.0-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
```

所以我们可以选择一个最大的数字来作为版本号安装。

```
# apt install -y kubelet=1.16.15-00 kubeadm=1.16.15-00 kubectl=1.16.15-00

... ...

Setting up kubelet (1.16.15-00) ...
Setting up kubectl (1.16.15-00) ...
Setting up kubeadm (1.16.15-00) ...
```

最后再使用 kubeadm 安装指定版本的 Kubernetes：

```
# kubeadm init --cri-socket /run/containerd/containerd.sock \
    --pod-network-cidr=10.244.0.0/16 \
    --apiserver-advertise-address 192.168.2.147 \
    --kubernetes-version v1.16.0
```
