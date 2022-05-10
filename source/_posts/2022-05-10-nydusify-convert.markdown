---
layout: post
title: "Nydus 源码解读：nydusify convert"
date: 2022-05-10 17:36:02 +0800
comments: true
categories: 
---

这是 nydus 源码学习的笔记，主要是介绍 nydusify convert 命令的流程，该命令主要用于转换 nydus 镜像，比如将 Docker Hub 上的镜像转换为 nydus 镜像，并保存到 registry。

本例使用了一个非常简单的例子，所需要的命令为：

```bash
$ nydusify  convert \
    --source liubin/nydus:two-layers \
    --target localhost:5000/test-nydus
```

其中源镜像 `liubin/nydus:two-layers` 是一个基于 alpine 的镜像，一共有两层。


## 启动 registry

转换后的 target 镜像会保存到自己创建的 registry 里，所以我们先在本地启动一个 registry 容器。

```bash
$ docker run -d -p 5000:5000 --restart=always --name registry registry:2
```

## 镜像转换流程

nydusify 命令转换镜像大概流程很简单：

- 解析源镜像，得的 manifest/config/layers 信息
- 下载每个 layer，并使用 nydus-image create 命令基于 OCI 镜像层目录进行转换，上传 bootstrap 和 blob 文件
- 组装 nydus 的 manifest 或 index 文件，上传到 target registry

这里我们就以 [v2.0.0-rc.5](https://github.com/dragonflyoss/image-service/tree/v2.0.0-rc.5) 为基础，看看具体的转换是怎么执行的。

nydusify 的代码在 `contrib/nydusify` 目录下，真正的、主要的执行在 contrib/nydusify/pkg/converter/converter.go 中的 [convert](https://github.com/dragonflyoss/image-service/blob/v2.0.0-rc.5/contrib/nydusify/pkg/converter/converter.go#L163-L376) 函数。这里是它的主要流程，我们删掉了一些不妨碍本文主旨的部分代码，比如 cache 相关的内容，以方面解说。

同时为方便阅读，解说也都以注释的方式放到了代码中。


```go

// 解析源镜像，获得一个 layer 数组
// 注意，这里镜像其实已经解析过了，此处只是基于解析的镜像从新组装了
// 一个 []SourceLayer{} 数组。
sourceLayers, err := sourceProvider.Layers(ctx)

// 两个 worker，一个用于 pull 源镜像，一个用于 push nydus 的 blob 和 bootstrap 。
pullWorker := utils.NewQueueWorkerPool(PullWorkerCount, uint(len(sourceLayers)))
pushWorker := utils.NewWorkerPool(PushWorkerCount, uint(len(sourceLayers)))

// build nydus 镜像的主要数据结构，对应镜像的一层
buildLayers := []*buildLayer{}

// Pull and mount source layer in pull worker
var parentBuildLayer *buildLayer
for idx, sourceLayer := range sourceLayers {
	buildLayer := &buildLayer{
		index:          idx,
		buildWorkflow:  buildWorkflow,
		bootstrapsDir:  bootstrapsDir,
		remote:         cvt.TargetRemote,
		source:         sourceLayer,
		parent:         parentBuildLayer,
		referenceBlobs: blobLayers,
	}
	// 由于镜像层有父子关系，所以这个关联关系还是需要在构建的时候传递下去的
	parentBuildLayer = buildLayer
	buildLayers = append(buildLayers, buildLayer)
	job := mountJob{
		ctx:   ctx,
		layer: buildLayer,
	}

	// 添加到 pull worker 队列
	if err := pullWorker.Put(&job); err != nil {
		return errors.Wrap(err, "Put layer pull job to worker")
	}
}

for _, jobChan := range pullWorker.Waiter() {
	select {
	case _job := <-jobChan:
		job := _job.(*mountJob)

		// 从队列中拿到一个 job ，然后调用 job 的 Build 方法，这也是 build 的主要执行体
		err := job.layer.Build(ctx)

		// 然后把这个 job 的 Push 方法添加到 push 队列
		pushWorker.Put(func() error {
			return job.layer.Push(ctx)
		})
	case err := <-pushWorker.Err():
		// 这里是为了快速拿到 push 错误后结束的特殊处理
	}
}

// 等待所有层 push 成功完成
if err := <-pushWorker.Waiter(); err != nil {
	return errors.Wrap(err, "Push Nydus layer in wait")
}

// Push OCI manifest, Nydus manifest and manifest index
mm := &manifestManager{
	sourceProvider: sourceProvider,
	remote:         cvt.TargetRemote,
}

if err := mm.Push(ctx, buildLayers); err != nil {
	// ... ...
}
```

这个函数虽然比较长，但是逻辑流程还是很清晰的。

下面我们再深入到细节看看具体不同步骤所完成的工作。

### 获取源镜像的 manifest 信息

这里是在[初始化 sourceProvider](https://github.com/dragonflyoss/image-service/blob/ae79d5a6ae9a29cf010fbce3b73dd0751f1bd52d/contrib/nydusify/pkg/converter/provider/source.go#L173-L205) 的时候做的：

```go
func DefaultSource(ctx context.Context, remote *remote.Remote, workDir, platform string) ([]SourceProvider, error) {

	parser, err := parser.New(remote, arch)
	parsed, err := parser.Parse(ctx)

	sp := []SourceProvider{
		&defaultSourceProvider{
			workDir: workDir,
			image:   *parsed.OCIImage,
			remote:  remote,
		},
	}

	return sp, nil
}
```

这里我们就不再深入解释 parser 相关代码，有兴趣可以参考[这里](https://github.com/dragonflyoss/image-service/blob/ae79d5a6ae9a29cf010fbce3b73dd0751f1bd52d/contrib/nydusify/pkg/parser/parser.go#L185-L264)。


我们的测试镜像 liubin/nydus:two-layers 没有 index，只有 manifest，它的解析过程以及信息如下面小节介绍。

#### 获取 image

请求 https://registry-1.docker.io/v2/liubin/nydus/manifests/two-layers ，获得数据为：

```json
{
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "digest": "sha256:00e2afc814fb9c3c48858dc5b17758e49ca2619f2bdd544c6f9810c397d83137",
  "size": 735
}
```

#### 获取 manifest

请求 https://registry-1.docker.io/v2/liubin/nydus/manifests/sha256:00e2afc814fb9c3c48858dc5b17758e49ca2619f2bdd544c6f9810c397d83137 ，得到该镜像的 manifest 信息：

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
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

这个文件很简单，说明该镜像由 1 个 config 对象和 2 个层 组成。

#### 获取 config 对象

请求 
https://registry-1.docker.io/v2/liubin/nydus/blobs/sha256:85f3e7735ea66cc3df14a38419b199913b84d616298fb38d1a4a5c92cf885d49

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

### “节外生枝”的镜像下载

一般来说，可能我们会按照解析镜像、下载镜像、转换镜像的步骤来做，但是 nydusify 镜像下载却稍微偏离了主处理流程，是在将 job 添加到 pullWorker 中的时候调用的。

具体来说， pullWorker 是一个 QueueWorkerPool 类型的队列，它所接受的任务必须是[这样](https://github.com/dragonflyoss/image-service/blob/ae79d5a6ae9a29cf010fbce3b73dd0751f1bd52d/contrib/nydusify/pkg/utils/worker.go#L14-L17)的数据结构：

```go
type RJob interface {
	Do() error
	Err() error
}
```

而在任务添加到这个队列的时候，会执行该任务的 [Do](https://github.com/dragonflyoss/image-service/blob/ae79d5a6ae9a29cf010fbce3b73dd0751f1bd52d/contrib/nydusify/pkg/utils/worker.go#L53-L60) 方法：

```go
job, ok := <-pool.jobs
// ... ...
err := job.Do()
```

而我们看到前面的 mountJob ，实际上就是 buildLayer 的一个包装：

```go
type mountJob struct {
	err    error
	ctx    context.Context
	layer  *buildLayer
	umount func() error
}
```

在这个任务添加到队列的时候，会执行 mountJob 的 Do() 方法：

```go
func (job *mountJob) Do() error {
	var umount func() error
	umount, job.err = job.layer.Mount(job.ctx)
	job.umount = umount
	return job.err
}
```

也就是说，在将任务添加到 pullWorker 队列的时候，实际上会调用 buildLayer 的 Mount() 方法。

```go

func (layer *buildLayer) Mount(ctx context.Context) (func() error, error) {
	bootstrapName := strconv.Itoa(layer.index+1) + "-" + layer.source.Digest().String()
	layer.bootstrapPath = filepath.Join(layer.bootstrapsDir, bootstrapName)

	mounts, umount, err := layer.source.Mount(ctx)

	layer.sourceMount, err = parseSourceMount(mounts)

	return umount, mountDone(nil)
}
```

mount 方法还会调用 provider.SourceLayer 的 Mount 方法，我们来看一下这个 Mount 方法。

```go

func (sl *defaultSourceLayer) Mount(ctx context.Context) ([]mount.Mount, func() error, error) {
	digestStr := sl.desc.Digest.String()

	if err := utils.WithRetry(func() error {
		reader, err := sl.remote.Pull(ctx, sl.desc, true)
		defer reader.Close()

		// Decompress layer from source stream
		if err := utils.UnpackTargz(ctx, sl.mountDir, reader); err != nil {
			return errors.Wrap(err, fmt.Sprintf("Decompress source layer %s", digestStr))
		}

		return nil
	}); err != nil {
		return nil, nil, err
	}

	umount := func() error {
		return os.RemoveAll(sl.mountDir)
	}

	mounts := []mount.Mount{
		{
			Type:   "oci-directory",
			Source: sl.mountDir,
		},
	}

	return mounts, umount, nil
}

```

我们可以看到，对于本例的 OCI 镜像来说，这里会执行真正的 pull 操作来下载一个 layer，然后解压缩，最后返回这个所谓的 mount 目录。其格式类似 `tmp/source/sha256:ce91720a155e2c10eba7a0a7e13a64efb22e3545b25f360e0cca426ee5881d59` 。


### 转换镜像

转换镜像的操作是按层进行的，主要处理是在 buildLayer（contrib/nydusify/pkg/converter/layer.go） 和 Workflow （contrib/nydusify/pkg/build/workflow.go） 和 Builder （contrib/nydusify/pkg/build/builder.go）中实现的。

简单来说，是 buildLayer 的 Build 方法会调用 Workflow 的 Build 方法，而 Workflow 的 Build 方法则会调用 Builder 的 Run 方法，Builder 最终调用 nydus-image 创建 nydus blob 和 bootstrap。

这中间虽然是层层调用，但多数为组装各种参数，我们直接看到最里层的 Builder 的 Run 函数。其实 Run 函数也很简单，就是组装调用 nydus-image 的参数，然后调用该命令创建 blob 和 bootstrap 文件。

我们就不看具体代码了，实际执行 nydus-image 的命令类似下面这样：

```go
$ nydus-image create \
  --parent-bootstrap tmp/bootstraps/1-sha256:a0d0a0d46f8b52473982a3c466318f479767577551a53ffc9074c9fa7035982e \
  --bootstrap tmp/bootstraps/2-sha256:f2afd747bba4e4e1ae68a143d8f610cc44ae712e988a50ef47c9dfb2fddbc290 \
  --log-level warn \
  --whiteout-spec oci \
  --output-json tmp/bootstraps/2-sha256:f2afd747bba4e4e1ae68a143d8f610cc44ae712e988a50ef47c9dfb2fddbc290-output.json \
  --blob tmp/blobs/d46cfc7b-1a5f-4101-b4bd-1016cf46979f \
  --prefetch-policy fs \
  tmp/source/sha256:ce91720a155e2c10eba7a0a7e13a64efb22e3545b25f360e0cca426ee5881d59 
```

我们看到，最后一个参数，也就是文件系统的目录，正好是前面 defaultSourceLayer 的 Mount 方法返回的 mount 位置。

### 上传 blob/bootstrap/manifest 文件

通过 nydus-image 为所有镜像层都创建完 blob 和 bootstrap 之后，就可以将这些构建物传到 target 里了，这里是我们在本地运行的一个 registry。

具体来说要上传的内包括：

- bootstrap
- blob
- manifest
- config
- index（如果有的话）

实现代码在 [contrib/nydusify/pkg/converter/manifest.go](https://github.com/dragonflyoss/image-service/blob/v2.0.0-rc.5/contrib/nydusify/pkg/converter/manifest.go#L197-L379) 文件中：


```go

func (mm *manifestManager) Push(ctx context.Context, buildLayers []*buildLayer) error {
	var (
		// 用于保存所有 blob 的 ID 列表，
		// 最后这个列表会保存到 bootstrap 的 
		// annotations["containerd.io/snapshot/nydus-blob-ids"] 里
		blobListInAnnotation []string
		referenceBlobs       []string
		layers               []ocispec.Descriptor
	)

	for idx, _layer := range buildLayers {
		record := _layer.GetCacheRecord()
		blobListInAnnotation = appendBlobs(blobListInAnnotation, layersHex(_layer.referenceBlobs))
		referenceBlobs = appendBlobs(referenceBlobs, layersHex(_layer.referenceBlobs))
		// try add reference blob layers to manifest
		if mm.backend.Type() == backend.RegistryBackend {
			for _, blobDesc := range _layer.referenceBlobs {
				if !containsLayer(layers, blobDesc.Digest) {
					layers = append(layers, blobDesc)
				}
			}
		}

		// Only need to write lastest bootstrap layer in nydus manifest
		if idx == len(buildLayers)-1 {
			// 将 blob ID 列表序列化后保存到 layer 的 annotation 里
			blobListBytes, err := json.Marshal(blobListInAnnotation)
			record.NydusBootstrapDesc.Annotations[utils.LayerAnnotationNydusBlobIDs] = string(blobListBytes)
			// bootstrap 层也加入 layers 数组
			layers = append(layers, *record.NydusBootstrapDesc)
		}
	}

	// 更新 OCI config 属性
	ociConfig, err := mm.sourceProvider.Config(ctx)
	ociConfig.RootFS.DiffIDs = []digest.Digest{}
	ociConfig.History = []ocispec.History{}

	for idx, desc := range layers {
		layerDiffID := digest.Digest(desc.Annotations[utils.LayerAnnotationUncompressed])
		if layerDiffID == "" {
			layerDiffID = desc.Digest
		}
		ociConfig.RootFS.DiffIDs = append(ociConfig.RootFS.DiffIDs, layerDiffID)
	}

	// 序列化 config 对象然后 push 到 registry
	configMediaType := ocispec.MediaTypeImageConfig
	configDesc, configBytes, err := utils.MarshalToDesc(ociConfig, configMediaType)

	if err := mm.remote.Push(ctx, *configDesc, true, bytes.NewReader(configBytes)); err != nil {
		return errors.Wrap(err, "Push Nydus image config")
	}

	// 构建 manifest 对象资源
	manifestMediaType := ocispec.MediaTypeImageManifest
	nydusManifest := struct {
		MediaType string `json:"mediaType,omitempty"`
		ocispec.Manifest
	}{
		MediaType: manifestMediaType,
		Manifest: ocispec.Manifest{
			Versioned: specs.Versioned{
				SchemaVersion: 2,
			},
			Config:      *configDesc,
			Layers:      layers,
			Annotations: mm.buildInfo.Dump(),
		},
	}

	// 序列化 manifest 对象资源
	nydusManifestDesc, manifestBytes, err := utils.MarshalToDesc(nydusManifest, manifestMediaType)
	if err != nil {
		return errors.Wrap(err, "Marshal Nydus image manifest")
	}

	// 设置 Platform 属性
	p, err := mm.CloneSourcePlatform(ctx, utils.ManifestOSFeatureNydus)
	nydusManifestDesc.Platform = p

	if !mm.multiPlatform {
		// 构建 manifest 对象资源
		if err := mm.remote.Push(ctx, *nydusManifestDesc, false, bytes.NewReader(manifestBytes)); err != nil {
			return errors.Wrap(err, "Push nydus image manifest")
		}
		return nil
	}

	// mm.multiPlatform == true 的话，则在 push manifest 后，还要构造 index 对象
	if err := mm.remote.Push(ctx, *nydusManifestDesc, true, bytes.NewReader(manifestBytes)); err != nil {
		return errors.Wrap(err, "Push nydus image manifest")
	}

	// 拿到老的 manifest
	ociManifestDesc, err := mm.sourceProvider.Manifest(ctx)

	// 添加 plotform 属性
	if ociManifestDesc != nil {
		p, err := mm.CloneSourcePlatform(ctx, "")
		ociManifestDesc.Platform = p
	}

	// 获取已有 manifest 列表
	existManifests, err := mm.getExistsManifests(ctx)
	// 根据 nydusManifestDesc 和 ociManifestDesc 来组装 index
	_index, err := mm.makeManifestIndex(ctx, existManifests, nydusManifestDesc, ociManifestDesc)
	indexMediaType := ocispec.MediaTypeImageIndex
	index := struct {
		MediaType string `json:"mediaType,omitempty"`
		ocispec.Index
	}{
		MediaType: indexMediaType,
		Index:     *_index,
	}

	// 序列化 index 后 push 到 registry
	indexDesc, indexBytes, err := utils.MarshalToDesc(index, indexMediaType)
	if err := mm.remote.Push(ctx, *indexDesc, false, bytes.NewReader(indexBytes)); err != nil {
	}

	return nil
}
```

这就是最后的 manifest push 流程。

方便理解，这里我们把 nydus 镜像的 manifest 和 config 一起 dump 出来方便对比观看。

manifest：

```json
{
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:9aed3854ac516fda6913a6e5de3f9d4dd5838f49b89fb956a94227585fdc8c7b",
    "size": 447
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.nydus.blob.v1",
      "digest": "sha256:d542d4a146037df4e940d6921f0e16e9e0307ea60090e71281655c16ce048f8a",
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
      "digest": "sha256:df520c0bae065336a455543acd7abd1ff6437dff7424dfdab327dd50576d6d59",
      "size": 18174,
      "annotations": {
        "containerd.io/snapshot/nydus-blob-ids": "[\"d542d4a146037df4e940d6921f0e16e9e0307ea60090e71281655c16ce048f8a\",\"0bb9d4bf5810ee24b997ce82186a6ec43c3b6e89ba0e1c8e9288157bb704cdd1\"]",
        "containerd.io/snapshot/nydus-bootstrap": "true"
      }
    }
  ],
  "annotations": {
    "nydus.trace.builder-version": "2.0.0-acaf55d8d6e4f6aecbbe2fc63443680623d998b2",
    "nydus.trace.nydusify-version": "acaf55d8d6e4f6aecbbe2fc63443680623d998b2.20220510.0340",
    "nydus.trace.source-digest": "sha256:00e2afc814fb9c3c48858dc5b17758e49ca2619f2bdd544c6f9810c397d83137",
    "nydus.trace.source-reference": "liubin/nydus:two-layers"
  }
}
```

config：

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
      "sha256:d542d4a146037df4e940d6921f0e16e9e0307ea60090e71281655c16ce048f8a",
      "sha256:0bb9d4bf5810ee24b997ce82186a6ec43c3b6e89ba0e1c8e9288157bb704cdd1",
      "sha256:1ad6993081526b32b2ba95d94d23808d91ec5f4bac546eb328d539336d66a34a"
    ]
  }
}
```

到这里，只是梳理了 nydusify convert 命令的处理流程，下一步，计划再进入到 nydus-image create 命令，看看它是如何将指定的一个层（文件夹）转换为 blob 和 bootstrap 的。

