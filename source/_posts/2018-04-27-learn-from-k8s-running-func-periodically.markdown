---
layout: post
title: "向 Kubernetes 学习 - 如何实现定时轮训"
date: 2018-04-27 19:29:04 +0800
comments: true
categories: 
---

编程中常见的场景包括定时执行任务，很多语言，比如 Java 提供了非常方便易用且功能强大的类库，但是 Golang 中并没有，虽然我们可以通过 `time.Ticker` 等机制实现，但是不如原生直接支持更简单。

其实参考别人的代码是最好的学习方式了，比如通过阅读 K8s 的源代码，我们可以学习到很多编程技巧。

这里我们以实现定时轮训功能为例，来看看 K8s 是怎么实现的。

首先简单介绍下 K8s 的简单架构，主要包括以下几个组件（大体流程）：

* API servier
* Scheduler
* Controller manager
* Kubelet

这几个组件只是完整运行 K8s 的4个组件，并不是全部，这里我们只是为了了解而只对这几个组件进行说明。

首先客户端（API 或者 kubectl 工具）发出请求，比如创建一个Deployment，Controller manager 会负责这个 Deployment 的生命周期，Scheduler 会为 Pod 分配节点，该节点上的 Kubelet 会负责在自己的机器上启动这个 Pod，这是一个非常简单的流程。

这些组件，都是以守护进程方式运行的，即会一直无限循环的运行下去，不会退出。在其中有很多例子都是诸如每隔 5 分钟做一次同步，每隔 10 秒做一次 retry 的尝试。对于这样的需求，即使只有一处，相信从代码结构的角度，我们都会把这些功能抽象成 util 函数。

K8s 就实现了很多这样的方法，比如 "k8s.io/apimachinery/pkg/util/wait" 包就有很多非常实用的方法，值得我们借鉴。

首先，最简单的每隔 10 秒执行一个函数，永不停止，那么可以用[这个方法](https://github.com/kubernetes/kubernetes/blob/release-1.10/staging/src/k8s.io/apimachinery/pkg/util/wait/wait.go#L78)：

```go
func Forever(f func(), period time.Duration)
```

如果你想在上面的基础上，在需要的时候停止循环，那么可以使用[下面的方法](https://github.com/kubernetes/kubernetes/blob/release-1.10/staging/src/k8s.io/apimachinery/pkg/util/wait/wait.go#L87)，增加一个用于停止的 `chan` 即可。

```go
func Until(f func(), period time.Duration, stopCh <-chan struct{})
```

上面的第三个参数 `stopCh` 就是用于退出无限循环的标志，停止的时候我们 close 掉这个 chan 就可以了。

K8s 的很多地方都用到了这些函数，比如这里在 [Container manager 启动的时候](https://github.com/kubernetes/kubernetes/blob/release-1.10/pkg/kubelet/dockershim/cm/container_manager_linux.go#L70-L81)，就启动了一个无限循环来进行实际的工作（ `m.doWork` 函数）


```go
func (m *containerManager) Start() error {
	// TODO: check if the required cgroups are mounted.
	if len(m.cgroupsName) != 0 {
		manager, err := createCgroupManager(m.cgroupsName)
		if err != nil {
			return err
		}
		m.cgroupsManager = manager
	}
	go wait.Until(m.doWork, 5*time.Minute, wait.NeverStop)
	return nil
}
```


上面只是简单的定期运行任务的例子，有时候，我们还会需要在运行前去检查先决条件，在条件满足的时候才去运行某一任务，这时候可以使用 [Poll 方法](https://github.com/kubernetes/kubernetes/blob/release-1.10/staging/src/k8s.io/apimachinery/pkg/util/wait/wait.go#L220)。

```go
func Poll(interval, timeout time.Duration, condition ConditionFunc)
```

这个函数会以 `interval` 为间隔，不断去检查 `condition` 条件是否为真，如果为真则可以继续后续处理；如果指定了 `timeout` 参数，则该函数也可以只常识指定的时间。

一个非常常见的例子就是我们在创建资源之后，资源不会立即就位，我们需要等资源创建完成之后，才能进行后续操作，这时候使用 `Poll` 方法应该就会非常方便了。

此外这个函数还有两个快捷方式， `PollImmediate` 、 `PollInfinite` 和 `PollImmediateInfinite` ，具体意义从名称上面即可理解。

[`PollUntil`](https://github.com/kubernetes/kubernetes/blob/release-1.10/staging/src/k8s.io/apimachinery/pkg/util/wait/wait.go#L289) 方法和上面的类似，但是没有 `timeout` 参数，多了一个 `stopCh` 参数。


此外，这个包里还有一个公开方法 [`WaitFor`](https://github.com/kubernetes/kubernetes/blob/release-1.10/staging/src/k8s.io/apimachinery/pkg/util/wait/wait.go#L307) ，好像并没有直接被外部调用，而是在 `PollUtil` 中被使用，这个方法需要自己编写 `WaitFunc` 类型的方法作为第一个参数，具体可以参考 `PollUntil` 的实现。 