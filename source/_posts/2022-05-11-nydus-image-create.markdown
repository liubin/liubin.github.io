---
layout: post
title: "Nydus 源码解读：nydus-image create"
date: 2022-05-11 08:31:47 +0800
comments: true
categories: 
---

上一篇 [Nydus 源码解读：nydusify convert](/blog/2022/05/10/nydusify-convert/) 介绍了 `nydusify convert` 命令的大致流程，我们在那一片已经看到，每一层镜像都是通过 `nydus-image` 命令来转换的。

这一篇我们就来以 `v2.0.0-rc.5` 的代码为基础，分析一下 `nydus-iamge create` 命令的代码。`nydus-image` 的代码主要在 `src/bin/nydus-image` 文件夹下。

## 测试环境

我们准备一个简单的文件夹 `fs`，然后将该文件夹转换为 nydus 镜像。

```bash
$ mkdir -p fs/dir1
$ mkdir -p fs/dir2
$ touch fs/foo.txt
$ echo abcde > fs/dir1/bar.txt

$ mkdir -p fs/dir1/dir1-1
$ touch fs/dir1/dir1-1/foo
$ echo abc > fs/dir1/dir1-1/hello

$ tree fs/
fs/
├── dir1
│   ├── bar.txt
│   └── dir1-1
│       ├── foo
│       └── hello
├── dir2
└── foo.txt

3 directories, 4 files
```

**注意**：这里我们都是使用了常规的文件，没有软硬链接、设备文件等特殊类型的文件。

然后我们使用如下命令来创建 nydus 镜像：

```bash
$ mkdir output
$ nydus-image create \
    --fs-version 5 \
    --bootstrap output/bootstrap \
    --blob-dir output \
    fs
```

上面参数的意思分别为：

- `--fs-version 5`：使用 Rafs 版本 5，目前支持 5/6，但是 6 应该还没有完全开发完成。
- `--bootstrap output/bootstrap`：bootstrap 文件的保存位置。
- `--blob-dir output`：blob 的保存位置
- `fs`：输入文件夹

实际上 nydus 支持从 3 种类型的输入源来创建 nydus 镜像：

- directory
- diff
- stargz_index

其中 stargz 也是一种镜像延迟加载技术。在本例中，我们使用了默认的 directory 方式。这个值可以通过命令行参数 `source-type` 来控制。

## 代码解析

下面我们就来看一下 nydus-image 的源代码。

### 主函数

nydus-image create 命令的入口在 [`src/bin/nydus-image/main.rs`](https://github.com/dragonflyoss/image-service/blob/v2.0.0-rc.5/src/bin/nydus-image/main.rs#L571-L702) 里，这里我们不贴代码了，只简单说一下里面干了什么。

这个函数主要创建了几个数据结构：

- build_ctx（类型：BuildContext）：主要在各函数调用中传递一些变量
- blob_mgr（类型：BlobManager）：管理 blob
- bootstrap_mgr（类型： BootstrapManager）：管理 bootstrap
- builder（类型：Box<dyn Builder>）：执行 build 过程

我们看到 builder 是一个接口，它会根据输入源类型的不同而初始化为不同的具体实现。在这个例子中，我们基于文件夹构建 Nydus 镜像，所以这里 builder 的实际类型行为：

```rust
builder = Box::new(DirectoryBuilder::new())
```

最后，该函数会调用 

```rust
builder.build(&mut build_ctx, &mut bootstrap_mgr, &mut blob_mgr)
```

来实现 build。

### builder.build() 函数

[builder.build() 函数](https://github.com/dragonflyoss/image-service/blob/v2.0.0-rc.5/src/bin/nydus-image/builder/directory.rs#L117-L196) 内容如下：


```rust

fn build(
    &mut self,
    ctx: &mut BuildContext,
    bootstrap_mgr: &mut BootstrapManager,
    blob_mgr: &mut BlobManager,
) -> Result<BuildOutput> {
    // 创建一个 BootstrapContext，在构建中将 bootstrap 数据保存在内存中
    let mut bootstrap_ctx = bootstrap_mgr.create_ctx(ctx.inline_bootstrap)?;

    // 基于输入源文件夹在内存中构建文件树结构
    let mut tree = self.build_tree_from_fs(ctx, &mut bootstrap_ctx, layer_idx)?;

    let mut bootstrap = Bootstrap::new()?;

    // 如果是多层构建，当层数据需要依赖父层进行计算
    if bootstrap_ctx.layered {
        // Merge with lower layer if there's one, do not prepare `prefetch` list during merging.
        bootstrap.build(ctx, &mut bootstrap_ctx, &mut tree)?;
        // 本例中不会走入这个分支，这里我们就不介绍 apply 方法了。
        tree = bootstrap.apply(ctx, &mut bootstrap_ctx, bootstrap_mgr, blob_mgr, None)?;
    }

    // 将树状结构转换为一个平铺的数组
    // 保存到 `bootstrap_ctx.nodes` 中
    timing_tracer!(
        { bootstrap.build(ctx, &mut bootstrap_ctx, &mut tree) },
        "build_bootstrap"
    )?;

    // Dump blob file
    let mut blob_ctx = BlobContext::new()?;
    let blob_index = blob_mgr.alloc_index()?;
    let mut blob = Blob::new();

    // 加载 builder 启动时指定的 chunk-dict 中的内容
    blob_mgr.extend_blob_table_from_chunk_dict()?;

    // blob.dump 将内存的数据写入磁盘 blob 文件
    let blob_exists = timing_tracer!(
        {
            blob.dump(
                ctx,
                &mut blob_ctx,
                blob_index,
                &mut bootstrap_ctx.nodes,
                &mut blob_mgr.chunk_dict_cache,
            )
        },
        "dump_blob"
    )?;

    let mut blob_writer = blob_ctx.writer.take().unwrap();
    let blob_id = blob_ctx.blob_id();

    // 这里的 blob_exists 表示 compressed_blob_size  > 0，即 blob 文件大小大于 0 
    if blob_exists {
        blob_writer.finalize(blob_id.clone())?;
        // Add new blob to blob table.
        blob_mgr.add(blob_ctx);
    }

    // 将 bootstrap 写入文件
    let blob_table = blob_mgr.to_blob_table(ctx)?;
    bootstrap.dump(ctx, &mut bootstrap_ctx, &blob_table)?;

    bootstrap_mgr.add(bootstrap_ctx);
}
```

`build_tree_from_fs` 函数也比较简单，就是构建文件夹结构，最后保存到 Tree 里：

```rust
pub(crate) struct Tree {
    /// Filesystem node.
    pub node: Node,
    /// Children tree nodes.
    pub children: Vec<Tree>,
}
```

其中一个 node 表示一个文件/文件夹，而 children 则表示该文件夹下的子文件夹或者文件。

有了文件夹的树状结构，bootstrap 就可以开始 build 了。

### Bootstrap build() 和 build_rafs 函数

[build()](https://github.com/dragonflyoss/image-service/blob/v2.0.0-rc.5/src/bin/nydus-image/core/bootstrap.rs#L57-L86) 函数时构建 bootstrap 的入口：


```rust
pub fn build(
    &mut self,
    ctx: &mut BuildContext,
    bootstrap_ctx: &mut BootstrapContext,
    tree: &mut Tree,
) -> Result<()> {
    // 设置为 root 节点
    tree.node.index = RAFS_ROOT_INODE;
    tree.node.inode.set_ino(RAFS_ROOT_INODE);

    bootstrap_ctx.inode_map.insert(
        (tree.node.layer_idx, tree.node.src_ino, tree.node.src_dev),
        vec![tree.node.index],
    );

    let mut nodes = Vec::with_capacity(0x10000);
    // 将根节点 push 进去
    nodes.push(tree.node.clone());

    // 调用 build_rafs 构建子节点
    self.build_rafs(ctx, bootstrap_ctx, tree, &mut nodes)?;

    // 将结果保存到 bootstrap_ctx.nodes 中
    bootstrap_ctx.nodes = nodes;

    Ok(())
}
```

[`build_rafs`](https://github.com/dragonflyoss/image-service/blob/v2.0.0-rc.5/src/bin/nydus-image/core/bootstrap.rs#L126-L252) 函数会遍历前面生成的节点列表，为各节点设置 inode 属性，包括 index，ino，子节点数量等。

这个函数会递归的进行构建，所以有一个参数 tree 表示当前节点，对于子节点，如果是文件夹的话，还会继续调用 build_rafs 函数，构建子节点的数据结构。

`nodes` 参数是一个全局的 Node 列表，这是一个将树状结构转换为一维数组的结构。

函数内容如下。

```rust
fn build_rafs(
    &mut self,
    ctx: &mut BuildContext,
    bootstrap_ctx: &mut BootstrapContext,
    tree: &mut Tree,
    nodes: &mut Vec<Node>,
) -> Result<()> {
    // 分配 index，新的值为 nodes 数组长度 + 1
    let index = nodes.len() as u32 + 1;
    // 每个 node 都会有这个 index 属性，方便从 nodes 数组获取该节点
    let parent = &mut nodes[tree.node.index as usize - 1];

    // Maybe the parent is not a directory in multi-layers build scenario, so we check here.
    if parent.is_dir() {
        parent.inode.set_child_index(index);
        parent.inode.set_child_count(tree.children.len() as u32);
    }

    let mut dirs: Vec<&mut Tree> = Vec::new();
    let parent_ino = parent.inode.ino();

    for child in tree.children.iter_mut() {
        let index = nodes.len() as u64 + 1;
        child.node.index = index;
        child.node.inode.set_parent(parent_ino);

        // 这里删除了 hardlink 处理的代码
        if !hardlink {
            // 分配 ino，也就是数组索引
            child.node.inode.set_ino(index);
            child.node.inode.set_nlink(1);
        }

        // 这里处理 whiteout 类型
        match (
            bootstrap_ctx.layered,
            child.node.whiteout_type(ctx.whiteout_spec),
        ) {
            (true, Some(whiteout_type)) => {
                // 这里省略 layered 的处理
            }
            (false, Some(whiteout_type)) => {
                // Remove overlayfs opaque xattr for single layer build
                if whiteout_type == WhiteoutType::OverlayFsOpaque {
                    child
                        .node
                        .remove_xattr(&OsString::from(OVERLAYFS_WHITEOUT_OPAQUE));
                }
                nodes.push(child.node.clone());
            }
            _ => {
                // 这个例子中走默认分支，
                // 将 child.node 添加到 nodes 数组
                nodes.push(child.node.clone());
            }
        }

        // 如果当前 child 是文件夹，还需要再次递归处理
        // 所以将该文件夹保存到 dirs 数组
        if child.node.is_dir() {
            dirs.push(child);
        }
    }

    // 循环完当前节点的 children 之后
    // 还需要再次递归处理子文件夹
    for dir in dirs {
        self.build_rafs(ctx, bootstrap_ctx, dir, nodes)?;
    }

}

```

经过上面的两个函数的处理，就将所有节点的数据保存到了 bootstrap_ctx.nodes 中，接着就可以生成 bootstrap 和 blob 文件了。

### blob.dump

blob 的 [dump()](https://github.com/dragonflyoss/image-service/blob/v2.0.0-rc.5/src/bin/nydus-image/core/blob.rs#L26-L82) 方法位于 `src/bin/nydus-image/core/blob.rs` 文件中。

其内容如下：

```rust
pub fn dump<'a, T: ChunkDict>(
    &mut self,
    ctx: &BuildContext,
    blob_ctx: &'a mut BlobContext,
    blob_index: u32,
    nodes: &mut Vec<Node>,
    chunk_dict: &mut T,
) -> Result<bool> {
    match ctx.source_type {
        SourceType::Directory | SourceType::Diff => {
            // layout_blob_simple 函数用于从 Vec<Node> 类型的 nodes
            // 返回 Vec<usize> 类型的 inodes
            // prefetch 相关的变量和预拉取有关，这里我们暂时忽略
            let (inodes, prefetch_entries) = blob_ctx
                .blob_layout
                .layout_blob_simple(&ctx.prefetch, nodes)?;

            // inodes 类型为 Vec<usize> ，inode 即 usize，
            // 用这个 inode 作为数组索引，从 nodes 数组获取 node 元素
            // 然后再调用 node 的 dump_blob 将 node 的信息写到磁盘
            for (idx, inode) in inodes.iter().enumerate() {
                let node = &mut nodes[*inode];
                let size = node
                    .dump_blob(ctx, blob_ctx, blob_index, chunk_dict)
                    .context("failed to dump blob chunks")?;
            }
        }
        SourceType::StargzIndex => {
            // 省略 stargz 处理的 case
        }
    }

    // 如果没有指定 blob_id，则自动从 blob_hash 创建 ID
    if blob_ctx.blob_id.is_empty() {
        blob_ctx.blob_id = format!("{:x}", blob_ctx.blob_hash.clone().finalize());
    }

    // compressed_blob_size > 0 才需要写到磁盘，
    // 这里返回的 blob_exists 即是该标志
    let blob_exists = blob_ctx.compressed_blob_size > 0;

    Ok(blob_exists)
}
```
### node.dump_blob

接着我们来看一下真正的写 blob 的函数，该函数位于 [src/bin/nydus-image/core/node.rs](https://github.com/dragonflyoss/image-service/blob/v2.0.0-rc.5/src/bin/nydus-image/core/node.rs#L316-L470) 文件中。

```rust
pub fn dump_blob<T: ChunkDict>(
    self: &mut Node,
    ctx: &BuildContext,
    blob_ctx: &mut BlobContext,
    blob_index: u32,
    chunk_dict: &mut T,
) -> Result<u64> {
    if self.is_dir() {
        return Ok(0);
    } else if self.is_symlink() {
        // 针对 symlink 的特殊处理
        return Ok(0);
    } else if self.is_special() {
        // 针对 special 文件的特殊处理
        return Ok(0);
    }

    let mut file = File::open(&self.path)
        .with_context(|| format!("failed to open node file {:?}", self.path))?;
    let mut inode_hasher = RafsDigest::hasher(ctx.digester);
    let mut blob_size = 0u64;

    // `child_count` of regular file is reused as `chunk_count`.
    // 一个文件过大的话，大于 chunk size，就要被拆分为多个 chunk 存储。
    for i in 0..self.inode.child_count() {
        let chunk_size = blob_ctx.chunk_size;
        let file_offset = i as u64 * chunk_size as u64;
        let chunk_size = if i == self.inode.child_count() - 1 {
            // 最后一个 chunk，可能实际的 chunk size 小于 blob_ctx.chunk_size
            // 这里检查下是否 chunk size 不足
            (self.inode.size() as u64)
                .checked_sub((chunk_size * i) as u64)
                .ok_or_else(|| {
                    anyhow!("the rest chunk size of inode is bigger than chunk_size")
                })? as u32
        } else {
            chunk_size
        };

        // 使用 read_exact 读取一个 chunk 的数据到 chunk_data
        let mut chunk_data = &mut blob_ctx.chunk_data_buf[0..chunk_size as usize];
        file.read_exact(&mut chunk_data)
            .with_context(|| format!("failed to read node file {:?}", self.path))?;

        // 生成 chunk_id
        let chunk_id = RafsDigest::from_buf(chunk_data, ctx.digester);
        inode_hasher.digest_update(chunk_id.as_ref());

        // 生成 chunk 对象，对本例来说这是一个 RafsV5ChunkInfo 类型的数据结构
        let mut chunk = self.inode.create_chunk();
        chunk.set_id(chunk_id);

        // 通过对比 chunk digest 来判断是否该 chunk 已经存在
        // 注意这里从两个缓存的地方获取，
        // 一个是 blob_ctx.chunk_dict，
        // 一个是输入参数的 chunk_dict
        let exist_chunk = match blob_ctx.chunk_dict.get_chunk(&chunk_id) {
            Some(v) => Some((v, true)),
            None => chunk_dict.get_chunk(&chunk_id).map(|v| (v, false)),
        };

        if let Some((cached_chunk, from_dict)) = exist_chunk {
            if cached_chunk.uncompressed_size() == 0
                || cached_chunk.uncompressed_size() == chunk_size
            {
                // 从 cached_chunk 拷贝数据
                chunk.copy_from(cached_chunk);
                chunk.set_file_offset(file_offset);
                if from_dict {
                    // 如果是 blob_ctx.chunk_dict 里已存在该 chunk，
                    // 则设置 blob index 为该 chunk 的 blob id
                    let idx = blob_ctx.chunk_dict.get_real_blob_idx(chunk.blob_index());
                    chunk.set_blob_index(idx);
                }

                let source = if from_dict {
                    ChunkSource::Dict
                } else {
                    ChunkSource::Build
                };
                self.chunks.push(NodeChunk {
                    source,
                    inner: chunk,
                });

                continue;
            }
        }

        // 没有从任何缓存中获得该 chunk，则创建该 chunk
        // 首先先对数据进行压缩处理
        let (compressed, is_compressed) = compress::compress(&chunk_data, ctx.compressor)
            .with_context(|| format!("failed to compress node file {:?}", self.path))?;
        let compressed_size = compressed.len();

        // 4k 对齐
        let aligned_chunk_size = if ctx.aligned_chunk {
            try_round_up_4k(chunk_size).unwrap()
        } else {
            chunk_size
        };

        // 更新 blob_ctx 中的一些信息
        let pre_decompress_offset = blob_ctx.decompress_offset;
        let pre_compress_offset = blob_ctx.compress_offset;

        blob_ctx.compress_offset += compressed_size as u64;
        blob_ctx.decompressed_blob_size =
            blob_ctx.decompress_offset + aligned_chunk_size as u64;
        blob_ctx.compressed_blob_size += compressed_size as u64;
        blob_ctx.decompress_offset += aligned_chunk_size as u64;
        blob_ctx.blob_hash.update(&compressed);

        // 将压缩后的 chunk 数据写入到 blob 文件
        if let Some(writer) = &mut blob_ctx.writer {
            writer
                .write_all(&compressed)
                .context("failed to write blob")?;
        }

        // 更新 chunk 对象的一些属性
        let chunk_index = blob_ctx.alloc_index()?;
        chunk.set_chunk_info(
            blob_index,
            chunk_index,
            file_offset,
            pre_decompress_offset,
            pre_compress_offset,
            compressed_size,
            chunk_size,
            is_compressed,
        )?;

        blob_ctx.add_chunk_meta_info(&chunk)?;

        // 将 chunk 对象加入到 chunk_dict，即该方法的输入参数
        chunk_dict.add_chunk(chunk.clone());

        // 将 chunk 对象添加到 self.chunks
        self.chunks.push(NodeChunk {
            source: ChunkSource::Build,
            inner: chunk,
        });
        blob_size += compressed_size as u64;
    }

    Ok(blob_size)
}
```

以上就是 dump blob 的大致流程。

### bootstrap.dump

Bootstrap 的 dump 会根据 rafs 版本来调用不同的实现函数，这里我们用的是 v5 的版本，所以实现函数在 [dump_rafsv5](https://github.com/dragonflyoss/image-service/blob/v2.0.0-rc.5/src/bin/nydus-image/core/bootstrap.rs#L373-L502) 中。


```rust
fn dump_rafsv5(
    &mut self,
    ctx: &mut BuildContext,
    bootstrap_ctx: &mut BootstrapContext,
    blob_table: &RafsV5BlobTable,
) -> Result<()> {
    // 计算文件夹 inode 的 digest
    for idx in (0..bootstrap_ctx.nodes.len()).rev() {
        self.digest_node(ctx, bootstrap_ctx, idx);
    }

    // Set inode table
    let super_block_size = size_of::<RafsV5SuperBlock>();
    let inode_table_entries = bootstrap_ctx.nodes.len() as u32;
    // 创建 inode table
    let mut inode_table = RafsV5InodeTable::new(inode_table_entries as usize);
    let inode_table_size = inode_table.size();

    // Set blob table, use sha256 string (length 64) as blob id if not specified
    let prefetch_table_offset = super_block_size + inode_table_size;
    let blob_table_offset = prefetch_table_offset + prefetch_table_size;
    let blob_table_size = blob_table.size();
    let extended_blob_table_offset = blob_table_offset + blob_table_size;
    let extended_blob_table_size = blob_table.extended.size();
    let extended_blob_table_entries = blob_table.extended.entries();

    // 创建 super block
    let mut super_block = RafsV5SuperBlock::new();
    let inodes_count = bootstrap_ctx.inode_map.len() as u64;
    super_block.set_inodes_count(inodes_count);
    super_block.set_inode_table_offset(super_block_size as u64);
    super_block.set_inode_table_entries(inode_table_entries);
    super_block.set_blob_table_offset(blob_table_offset as u64);
    super_block.set_blob_table_size(blob_table_size as u32);
    super_block.set_extended_blob_table_offset(extended_blob_table_offset as u64);
    super_block.set_extended_blob_table_entries(u32::try_from(extended_blob_table_entries)?);
    super_block.set_prefetch_table_offset(prefetch_table_offset as u64);
    super_block.set_prefetch_table_entries(prefetch_table_entries);
    super_block.set_compressor(ctx.compressor);
    super_block.set_digester(ctx.digester);
    super_block.set_chunk_size(ctx.chunk_size);

    // Set inodes and chunks
    let mut inode_offset = (super_block_size
        + inode_table_size
        + prefetch_table_size
        + blob_table_size
        + extended_blob_table_size) as u32;

    for node in &mut bootstrap_ctx.nodes {
        inode_table.set(node.index, inode_offset)?;
        // Add inode size
        inode_offset += node.inode.inode_size() as u32;
        // Add chunks size
        if node.is_reg() {
            inode_offset += node.inode.child_count() * size_of::<RafsV5ChunkInfo>() as u32;
        }
    }

    // Dump super block
    super_block
        .store(bootstrap_ctx.writer.as_mut())
        .context("failed to store superblock")?;

    // Dump inode table
    inode_table
        .store(bootstrap_ctx.writer.as_mut())
        .context("failed to store inode table")?;

    // Dump blob table
    blob_table
        .store(bootstrap_ctx.writer.as_mut())
        .context("failed to store blob table")?;

    // Dump extended blob table
    blob_table
        .store_extended(bootstrap_ctx.writer.as_mut())
        .context("failed to store extended blob table")?;

    // Dump inodes and chunks
    timing_tracer!(
        {
            for node in &bootstrap_ctx.nodes {
                node.dump_bootstrap_v5(&ctx, bootstrap_ctx.writer.as_mut())
                    .context("failed to dump bootstrap")?;
            }

            Ok(())
        },
        "dump_bootstrap",
        Result<()>
    )?;

    bootstrap_ctx
        .writer
        .finalize(Some(bootstrap_ctx.name.to_string()))?;

    Ok(())
}
```

关于 bootstrap 数据中存储的内容，可以参考前几天的 [这篇文章](/blog/2022/04/27/nydus-bootstrap-layout/)。

## 结束

Nydus 的代码量还是挺大，上面只是分析了大致的流程，至于具体到其中的每一个函数调用，可能还涉及到很多不同的执行路径，不同的标志值，肯定要比上面的复杂的多，到时候还得根据实际情况，进一步的细读。

