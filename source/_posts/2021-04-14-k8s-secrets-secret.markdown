---
layout: post
title: "关于 K8s 的 Secret 并不安全这件事"
date: 2021-04-14 15:56:47 +0800
comments: true
categories: 
---

K8s 提供了 Secret 资源供我们来保存、设置一些敏感信息，比如 API endpoint 地址，各种用户密码或 token 之类的信息。在没有使用 K8s 的时候，这些信息可能是通过配置文件或者环境变量在部署的时候设置的。

不过，Secret 其实并不安全，稍微用 kubectl 查看过 Secret 的人都知道，我们可以非常方便的看到 Secret 的原文，只要有相关的权限即可，尽管它的内容是 base64 编码的，这基本上等同于明文。

所以说，K8s 原生的 Secret 是非常简单的，不是特别适合在大型公司里直接使用，对 RBAC 的挑战也比较大，很多不该看到明文信息的人可能都能看到。

尤其是现在很多公司采用了所谓的 GitOps 理念，很多东西都需要放到 VCS，比如 git 中，这一问题就更日益突出，因为 VCS 也得需要设置必要的权限。

## 问题

简单来说，大概有几个地方都可以让不应该看到 Secret 内容的人获得 Secret 内容：

- etcd 存储
- 通过 API server
- 在 node 上直接查看文件

这里我们以这个例子来看一下 Secret 在 K8s 中的使用情况。

Secret 定义， 用户名和密码分别为 `admin` 和 `hello-secret`：

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: aGVsbG8tc2VjcmV0Cg==

```

Pod 定义，这里我们将 Secret 作为 volume mount 到容器中。

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: docker.io/containerstack/alpine-stress
    command:
      - top
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

Pod 启动后，我们可以到容器中来查看 Secret 作为 volume mount 到容器后的文件内容。

```
$ kubectl exec -it mypod sh
/ # cd /etc/foo/
/etc/foo # ls -tal
total 4
drwxr-xr-x    1 root     root          4096 Apr 14 08:55 ..
drwxrwxrwt    3 root     root           120 Apr 14 08:55 .
drwxr-xr-x    2 root     root            80 Apr 14 08:55 ..2021_04_14_08_55_54.401661151
lrwxrwxrwx    1 root     root            31 Apr 14 08:55 ..data -> ..2021_04_14_08_55_54.401661151
lrwxrwxrwx    1 root     root            15 Apr 14 08:55 password -> ..data/password
lrwxrwxrwx    1 root     root            15 Apr 14 08:55 username -> ..data/username
/etc/foo # ls -tal ..2021_04_14_08_55_54.401661151
total 8
drwxr-xr-x    2 root     root            80 Apr 14 08:55 .
drwxrwxrwt    3 root     root           120 Apr 14 08:55 ..
-rw-r--r--    1 root     root            13 Apr 14 08:55 password
-rw-r--r--    1 root     root             5 Apr 14 08:55 username
/etc/foo # cat password
hello-secret
/etc/foo # 
```

### etcd 存储

API server 中的资源都保存在 etcd 中，我们可以直接从文件看到相关内容：

```
# hexdump -C /var/lib/etcd/member/snap/db | grep -A 5 -B 5 hello
00043640  12 00 1a 07 64 65 66 61  75 6c 74 22 00 2a 24 32  |....default".*$2|
00043650  35 66 37 35 38 30 38 2d  37 33 31 33 2d 34 38 64  |5f75808-7313-48d|
00043660  39 2d 39 61 38 65 2d 38  61 35 66 66 32 32 63 64  |9-9a8e-8a5ff22cd|
00043670  64 35 39 32 00 38 00 42  08 08 98 dc da 83 06 10  |d592.8.B........|
00043680  00 7a 00 12 19 0a 08 70  61 73 73 77 6f 72 64 12  |.z.....password.|
00043690  0d 68 65 6c 6c 6f 2d 73  65 63 72 65 74 0a 12 11  |.hello-secret...|
000436a0  0a 08 75 73 65 72 6e 61  6d 65 12 05 61 64 6d 69  |..username..admi|
000436b0  6e 1a 06 4f 70 61 71 75  65 1a 00 22 00 00 00 00  |n..Opaque.."....|
000436c0  00 00 00 08 95 5f 00 00  00 00 00 00 00 00 0a 37  |....._.........7|
000436d0  2f 72 65 67 69 73 74 72  79 2f 73 65 72 76 69 63  |/registry/servic|
000436e0  65 73 2f 65 6e 64 70 6f  69 6e 74 73 2f 6b 75 62  |es/endpoints/kub|
```

可以看到，基本 yaml 中的内容都是明文存放的，而且是进行 base64 解码之后的内容。

使用下面的命令也可以获得类似的结果。

```
$ ETCDCTL_API=3 etcdctl get --prefix /registry/secrets/default/mysecret | hexdump -C
```

etcd 本来存储的是明文数据，这个好像已经从 1.7 开始支持 [加密存储](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) 了，而且直接访问 etcd 从物理上来说也不是那么容易。

### API server

通过 API server 则简单的多，只要有权限就可以从任何节点上通过访问 API server 来得到 secret 的明文内容。

```
$ kubectl get secret mysecret -o yaml

apiVersion: v1
data:
  password: aGVsbG8tc2VjcmV0Cg==
  username: YWRtaW4=
kind: Secret
metadata:
  creationTimestamp: "2021-04-14T08:55:52Z"
  name: mysecret
  namespace: default
  resourceVersion: "2196"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: 25f75808-7313-48d9-9a8e-8a5ff22cdd59
type: Opaque

```

### 节点上

在节点上也可以看到 Secret 文件的内容。

查找 foo volume 的挂载点：

```
# mount | grep foo
tmpfs on /var/lib/kubelet/pods/280451e8-512b-489c-b5dd-df2b1a3c9b29/volumes/kubernetes.io~secret/foo type tmpfs (rw,relatime)
```

查看这个 volume 下面的文件内容：

```
# ls -tal /var/lib/kubelet/pods/280451e8-512b-489c-b5dd-df2b1a3c9b29/volumes/kubernetes.io~secret/foo
total 4
drwxrwxrwt 3 root root  120 4月  14 16:55 .
drwxr-xr-x 2 root root   80 4月  14 16:55 ..2021_04_14_08_55_54.401661151
lrwxrwxrwx 1 root root   31 4月  14 16:55 ..data -> ..2021_04_14_08_55_54.401661151
lrwxrwxrwx 1 root root   15 4月  14 16:55 password -> ..data/password
lrwxrwxrwx 1 root root   15 4月  14 16:55 username -> ..data/username
drwxr-xr-x 4 root root 4096 4月  14 16:55 ..

# cat /var/lib/kubelet/pods/280451e8-512b-489c-b5dd-df2b1a3c9b29/volumes/kubernetes.io~secret/foo/password
hello-secret
```

## 第三方方案

针对上面提到的可能泄露 Secret 的几点，大概解决方案不难想到如下几种：

- etcd 加密
- API server 严格进行权限设计
- 强化 node 节点用户权限管理和系统安全

不过，要相关确保 Secret 的绝对安全，上面这几种方案都是必须，缺一不可，缺少了那一个都相当于在墙上留了一个洞。

社区和公有云提供商都有一些产品和方案，我们可以参考一下。

- [shyiko/kubesec](https://github.com/shyiko/kubesec): Secure Secret management for Kubernetes (with gpg, Google Cloud KMS and AWS KMS backends)
- [bitnami-labs/sealed-secrets](https://github.com/bitnami-labs/sealed-secrets): A Kubernetes controller and tool for one-way encrypted Secrets
- [Vault by HashiCorp](https://www.vaultproject.io/docs/platform/k8s)
- [mozilla/sops](https://github.com/mozilla/sops)
- [Kubernetes External Secrets](https://github.com/external-secrets/kubernetes-external-secrets)
- [Kamus](https://github.com/Soluto/kamus)

### shyiko/kubesec

kubesec 只对 Secret 中数据进行加密/解密，支持如下 key 管理服务或软件：

- AWS Key Management Service
- Google Cloud KMS
- GnuPG

### bitnami-labs/sealed-secrets

Bitnami 在 K8s 领域也是一家人人知晓的公司，输出了很多技术和最佳实践。

![](/images/2021/sealed-secrets.png)

*本图来自 [Sealed Secrets: Protecting your passwords before they reach Kubernetes](https://engineering.bitnami.com/articles/sealed-secrets.html)*

SealeSecret 将 secret 资源整个加密保存为 `SealedSecret` 资源，而解密只能由该集群中的 controller 进行。

SealeSecret 提供了一个 kubeseal 工具来对 secret 资源进行加密，这个过程需要一个公开 key（公钥），这个公开 key 就是从 SealeSecret controller 拿到的。

不过，只从从说明文档来看， SealeSecret controller 加密解密所依赖的 key，也是通过普通的 Secret 来保存的，这难道不是一个问题？同时也增加了 SealeSecret controller 的运维成本。

### mozilla/sops

严格来说， sops 跟 K8s 并没有什么必然关系，它只是一个支持 YAML/JSON/ENV/INI 等文件格式的加密文件编辑器，它支持 AWS KMS, GCP KMS, Azure Key Vault, age, 和 PGP 等服务与应用。

如果有有兴趣可以看它的[主页](https://github.com/mozilla/sops)。

### Kubernetes External Secrets

Kubernetes External Secrets 是知名域名服务提供商 godaddy 开发的开源软件，它可以直接将保存在外部 KMS 中的机密信息传给 K8s 。目前支持的 KSM 包括：

- AWS Secrets Manager
- AWS System Manager
- Hashicorp Vault
- Azure Key Vault
- GCP Secret Manager
- Alibaba Cloud KMS Secret Manager

它通过自定义 controller 和 CRD 来实现，具体架构图如下：

![](/images/2021/kubernetes-external-secrets-architecture.png)

具体来说用户需要创建一个 ExternalSecret 类型的资源，来将外部 KMS 的数据映射到 K8s 的 Secret 上。

不过，这个中方式大概只有两点好处：

- 统一 key 的管理，或者沿用既有 key 资产
- key 信息不想放到 VCS 等

对于防止 Sercet 信息泄露，作用不大，因为其明文资源还是可以在 API server/etcd 上看到。

或者说，External Secrets 真正做的事情，也就是从外部 KMS 中的 key ，映射成 K8s 中的 Secret 资源而已，对保证在 K8s 集群中数据的安全性用处不大。

### Kamus

Kamus 同样提供了加密 key 的方法（一个命令行工具），同时只有通过 K8s 中的 controller 才能对这个 key 进行解密。不过它 保存在 K8s 中的 Secret 是加密的状态，用户不能像 External Secrets 那样直接获得 Secret 的明文内容。

Kamus 由 3 个组件组成，分别是：

- Encrypt API
- Decrypt API
- Key Management System (KMS)

KMS 是一个外部加密服务的封装，目前支持如下服务：
- AES
- AWS KMS
- Azure KeyVault 
- Google Cloud KMS

Kamus 以 service account 为单位对 secret 进行加密，之后 Pod 会通过 service account 来请求 Kamus 的解密服务来对该 secret 进行解密。

对 K8s 来说，解密 secret 可以通过 init container 来实现：定义一个 基于内存的 emptyDir ，业务容器和 init 容器使用同一个 volume， init 容器解密后，将数据存放到该 volume 下，之后业务容器就可以使用解密后的 secret 数据了。

![](/images/2021/kamus-pod.png)

### Vault by HashiCorp

HashiCorp 公司就不多说，在云计算/DevOps领域也算是数一数二的公司了。

Vault 本身就是一个 KMS 类似的服务，用于管理机密数据。对于 K8s 的原生 secret ，大概提供了如下两种方式的支持：

- [Agent Sidecar Injector/vault-k8s](https://www.vaultproject.io/docs/platform/k8s/injector)
- [Vault CSI Provider](https://www.vaultproject.io/docs/platform/k8s/csi)

#### Agent Sidecar Injector

这种方式和上面的 Kamus 类似，也是需要两个组件：

- Mutation webhook：负责修改 pod 定义，注入init/sidecar
- agent-sidecar：负责获取和解密数据，将数据保存到指定的 volume/路径下

Vault agent sidecar injector 不仅提供了 init container 来初始化 secret ，还通过 sidecar 来定期更新 secret ，这样就非常接近原生 secret 的实现了。

应用程序则只需要在文件系统上读取指定的文件就可以了，而不必关系如何从外部获取加密信息。

这是官方 blog 中的一个示例：

Pod 信息：

```
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-secret-helloworld: "secrets/helloworld"
        vault.hashicorp.com/role: "myapp"
```

这个定义中，`vault-k8s` 会对该 pod 注入 `vault agent`，并使用 `secrets/helloworld` 来初始化。Pod 运行后，可以在 `/vault/secrets` 下找到一个名为 `helloworld` 的文件。

```
$ kubectl exec -ti app-XXXXXXXXX -c app -- cat /vault/secrets/helloworld
data: map[password:foobarbazpass username:foobaruser]
metadata: map[created_time:2019-12-16T01:01:58.869828167Z deletion_time: destroyed:false version:1]
```

当然这个数据是raw data，没有经过格式化。如果想指定输出到文件中的格式，可以使用 vault 的模板功能。

#### Vault CSI Provider

这部分可以参考下面的社区方案部分。

## 社区方案

当然，社区没有理由意识不到原生 secret 的问题，因此社区也有 [Kubernetes Secrets Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) ，一种通过 CSI 接口将 Secret 集成到 K8s 的方案。

Secrets Store CSI driver（`secrets-store.csi.k8s.io`）可以让 K8s mount 多个 secret 以 volume 的形式，从外部 KMS mount 到 Pod 里。

要想使用 Secrets Store CSI Driver ，大致过程如下:

- 定义 SecretProviderClass

```
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: my-provider
spec:
  provider: vault   # accepted provider options: azure or vault or gcp
  parameters:       # provider-specific parameters
```

- 为 Pod 配置 Volume

```
kind: Pod
apiVersion: v1
metadata:
  name: secrets-store-inline
spec:
  containers:
  - image: k8s.gcr.io/e2e-test-images/busybox:1.29
    name: busybox
    command:
    - "/bin/sleep"
    - "10000"
    volumeMounts:
    - name: secrets-store-inline
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
    - name: secrets-store-inline
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: "my-provider"
```

Pod 启动之后，就可以确认解密后的数据了：

```
$ kubectl exec secrets-store-inline -- ls /mnt/secrets-store/
foo
```

## 总结

上面的总结都是基于互联网公开的资料而来，并没有经过亲身体验，因此有些地方可能理解有误，要想深入了解还需要自己亲手确认最好。

不过总体来说，社区这种方案可能最简单，部署也不是很麻烦，只是这就和原生的 secret 基本没什么关系了。。。

Vault 方案也很成熟，值得关注。
