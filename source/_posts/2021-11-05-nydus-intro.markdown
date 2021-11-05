---
layout: post
title: "初试 Nydus"
date: 2021-11-05 16:49:13 +0800
comments: true
categories: 
---

[Nydus](https://github.com/dragonflyoss/image-service) 是一个镜像加速方案，用于解决大规模集群中镜像部署的性能问题。

在多数情况下，镜像尽管很大，但是里面的文件不一定全会在容器运行中使用。尤其是 Docker 多层镜像机制中，即使在上层被删除的文件，在底层还是存在的，拉取镜像的时候，这部分数据也会造成无谓的浪费。

最大的问题在于，容器必须在镜像下载到本地之后，才能启动容器。如果镜像没有优化，会大幅度降低镜像启动速度。 Nydus 的想法是容器启动时并不真正的把所有文件都下载到本地，而是在容器真的读取该文件时，才会从网络下载该文件（块），这样就可以减少容器启动的时间。

这篇文章不打算深入了解它的结构，只是先体验一下它是如何管理镜像的。

Nydus 采用 RAFS 文件系统，以 FUSE 形式提供给容器运行时使用。Nydus 数据分为两类：

- bootstrap：RAFS 的 metadata
- blob：存储真正的数据块

## 转换镜像

我们这里使用了一个非常简单的 2 层的镜像：

```
# Dockerfile
FROM docker.io/library/alpine:latest
ADD readme.txt /
```

基于这个 Dockerfile 构建后 push 到 Docker Hub.

然后可以使用 `nydusify` 命令将标准的 OCI 镜像转换为 Nydus 格式的镜像：

```bash
# nydusify convert \
  --source docker.io/liubin/nydus:two-layers \
  --target liubin/nydus:two-layers-nydus \
  -log-level debug
```

该命令会将源镜像转换成 Nydus 的格式，并存储到 liubin/nydus:two-layers-nydus 中。

Nydus 的 blob 存储可以选择 Registry 和阿里云 OSS 两种方式。默认使用 Image Retistry。

默认情况下临时文件会保存到 `./tmp` 下面：

```bash
# tree tmp/
tmp/
├── blobs
├── bootstraps
│   ├── 1-sha256:a0d0a0d46f8b52473982a3c466318f479767577551a53ffc9074c9fa7035982e
│   ├── 1-sha256:a0d0a0d46f8b52473982a3c466318f479767577551a53ffc9074c9fa7035982e-output.json
│   ├── 2-sha256:f2afd747bba4e4e1ae68a143d8f610cc44ae712e988a50ef47c9dfb2fddbc290
│   └── 2-sha256:f2afd747bba4e4e1ae68a143d8f610cc44ae712e988a50ef47c9dfb2fddbc290-output.json
├── conversion_metrics.prom
└── source

```

`tmp/bootstraps/2-sha256\:f2afd747bba4e4e1ae68a143d8f610cc44ae712e988a50ef47c9dfb2fddbc290-output.json`:

```json
{
  "blobs": [
    "66127347ad6d346cef12a6fdef2bb4ca611ff0f918b235136453da45bba83da9",
    "0bb9d4bf5810ee24b997ce82186a6ec43c3b6e89ba0e1c8e9288157bb704cdd1"
  ],
  "trace": {
    "consumed_time": {
      "apply layers": 6.585999926755903e-06,
      "build rafs": 0.0015352850314229727,
      "dump bootstrap and blob": 0.0024188628885895014,
      "load nodes from local directory": 3.126099909422919e-05,
      "load nodes from parent": 0.001709955045953393,
      "total build time": 0.005979558918625116,
      "validate bootstrap out of band": 0.002757467096671462,
      "write all nodes to blob including hashing": 0.0011135309468954802,
      "write all nodes to bootstrap": 0.0011579629499465227
    },
    "registered_events": {
      "blob compressed size": 0,
      "egid": "0",
      "euid": "0",
      "loading files from directory": 0,
      "loading files from parent": 1125
    }
  }
}
```

## 验证镜像

```bash
# nydusify check \
  --source docker.io/liubin/nydus:two-layers \
  --target liubin/nydus:two-layers-nydus
```

该命令会下载 OCI 镜像和 Nydus 镜像的 metadata 等数据，我们可以通过对比这两种不同类型的镜像的文件看一下看一下 Nydus 是如何保存镜像的。

默认情况下该命令会将结果都写到 `./output` 文件：

```bash
# tree output/
output/
├── fs
│   ├── nydus_api.sock
│   ├── nydus_blobs
│   ├── nydusd_config.json
│   ├── nydus_mounted
│   └── source_mounted
├── nydus_bootstrap
├── nydus_bootstrap_debug.json
├── nydus_config.json
├── nydus_manifest.json
├── oci_config.json
└── oci_manifest.json

4 directories, 8 files

```

我们这里只来看一下 OCI 和 Nydus 两种镜像的 manifest 文件。

首先看一下 OCI 的：

`oci_manifest.json`:

```json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "digest": "sha256:85f3e7735ea66cc3df14a38419b199913b84d616298fb38d1a4a5c92cf885d49",
    "size": 1674
  },
  "layers": [
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "digest": "sha256:a0d0a0d46f8b52473982a3c466318f479767577551a53ffc9074c9fa7035982e",
      "size": 2814446
    },
    {
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "digest": "sha256:f2afd747bba4e4e1ae68a143d8f610cc44ae712e988a50ef47c9dfb2fddbc290",
      "size": 123
    }
  ]
}
```

`oci_config.json`:

```json
{
  "created": "2021-11-05T07:39:26.692325137Z",
  "architecture": "amd64",
  "os": "linux",
  "config": {
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh"
    ]
  },
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:e2eb06d8af8218cfec8210147357a68b7e13f7c485b991c288c2d01dc228bb68",
      "sha256:f66394ac63df4047b5f85b83ace284471b145ca9d0c029e204318c5641b3fa43"
    ]
  },
  "history": [
    {
      "created": "2021-08-27T17:19:45.553092363Z",
      "created_by": "/bin/sh -c #(nop) ADD file:aad4290d27580cc1a094ffaf98c3ca2fc5d699fe695dfb8e6e9fac20f1129450 in / "
    },
    {
      "created": "2021-08-27T17:19:45.758611523Z",
      "created_by": "/bin/sh -c #(nop)  CMD [\"/bin/sh\"]",
      "empty_layer": true
    },
    {
      "created": "2021-11-05T07:39:26.692325137Z",
      "created_by": "/bin/sh -c #(nop) ADD file:d46e26bb98315616de9963c6399d128eabb29c54472e7633417d95a13ee6a287 in / "
    }
  ]
}
```

我们可以看到 OCI 镜像的 manifest 中有两层文件，每一层都是 `application/vnd.docker.image.rootfs.diff.tar.gzip` 类型，也是 Docker Hub 存储的 Blob。

再来看一下 Nydus 的镜像 manifest ：

`nydus_manifest.json`:

```json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:2f47f937739061383e02392a922b317f8865a7cb0d49643a8bf0c042380bcc34",
    "size": 447
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.nydus.blob.v1",
      "digest": "sha256:66127347ad6d346cef12a6fdef2bb4ca611ff0f918b235136453da45bba83da9",
      "size": 3687032,
      "annotations": {
        "containerd.io/snapshot/nydus-blob": "true"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.layer.nydus.blob.v1",
      "digest": "sha256:0bb9d4bf5810ee24b997ce82186a6ec43c3b6e89ba0e1c8e9288157bb704cdd1",
      "size": 17,
      "annotations": {
        "containerd.io/snapshot/nydus-blob": "true"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:8ca1c17af7e6230b522cea7e12ff7262a63331a09ae895cfb5fca3b6d1631a45",
      "size": 17819,
      "annotations": {
        "containerd.io/snapshot/nydus-blob-ids": "[\"66127347ad6d346cef12a6fdef2bb4ca611ff0f918b235136453da45bba83da9\",\"0bb9d4bf5810ee24b997ce82186a6ec43c3b6e89ba0e1c8e9288157bb704cdd1\"]",
        "containerd.io/snapshot/nydus-bootstrap": "true"
      }
    }
  ],
  "annotations": {
    "nydus.trace.nydusify-version": "2f10c409cb302eb63e29363e330bc00e89a39c8b.20211027.0608",
    "nydus.trace.source-digest": "sha256:00e2afc814fb9c3c48858dc5b17758e49ca2619f2bdd544c6f9810c397d83137",
    "nydus.trace.source-reference": "docker.io/liubin/nydus:two-layers"
  }
}
```

`nydus_config.json`:

```json
{
  "created": "2021-11-05T07:39:26.692325137Z",
  "architecture": "amd64",
  "os": "linux",
  "config": {
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh"
    ]
  },
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:66127347ad6d346cef12a6fdef2bb4ca611ff0f918b235136453da45bba83da9",
      "sha256:0bb9d4bf5810ee24b997ce82186a6ec43c3b6e89ba0e1c8e9288157bb704cdd1",
      "sha256:227fce4a699330352477cc3569e37459151e4e6cde2a38c5248c58053be34c9d"
    ]
  }
}
```

从 `nydus_manifest.json` 文件我们可以看到以下一些变化：

- OCI 的 2 层镜像变成了 3 层，这是因为多了一层 `"containerd.io/snapshot/nydus-bootstrap": "true"` 的 bootstrap 层
- Blob 层类型为 `application/vnd.oci.image.layer.nydus.blob.v1` ，且增加了一个 `"containerd.io/snapshot/nydus-blob": "true"` 的 annotation。
- bootstrap 层里 `containerd.io/snapshot/nydus-blob-ids` 保存了它所依赖的 Blobs。

今天的了解就到这里为止，下次再见。