---
layout: post
title: "Nydus bootstrap 文件解析"
date: 2022-04-27 10:54:21 +0800
comments: true
categories: 
---

这是零零散散的关于 Nydus 学习笔记中的一段，主要是了解下 bootstrap 文件的存储格式。我们知道 nydus 会创建两种类型的文件：

- bootstrap： 存储文件系统的元数据
- blob： 存储数据内容

这里我们以一个很简单的例子，来看一下 bootstrap 都保存了哪些文件系统的信息。

## 环境准备

我们使用的测试例子很简单，通过 nydus-image create 创建一个 bootstrap，这个 bootstrap 的输入文件夹内容也很少。

```bash
$ mkdir fs
$ touch fs/aaa
$ date > fs/bbb
$ nydus-image create --fs-version 5 --bootstrap output/bootstrap --blob-dir output fs
```

可以看出，该文件系统一共有两个文件：一个空文件，一个很简单的文本文件。

```bash
$ ls -l output/bootstrap 
-rw-r--r-- 1 vagrant vagrant 8832 Apr 26 06:55 output/bootstrap
```

## bootstrap 文件解析

我们使用 `xxd` 命令来输出 bootstrap 文件的 16 进制内容，具体命令如下：

```bash
$ xxd -a -e output/bootstrap
```

`xxd` 比 `hexdump` 的优点是可以非常简单的输出 `little-endian` 的内容，即其 `-e` 选项，rafs super block 存储的时候使用的就是 little-endian 的字节顺序。`-a` 选项用于去除空行（全 0x00 的行）。`-g 1` 选项设置一个字节单独一个字段，可以帮助我们计数，默认的话一个字段 4 个字节，即一个 u32 。

而且 `xxd` 输出默认就是 32 位一个字段，正好是一个 u32 ，而 `RafsV5SuperBlock` 中很多数据就是 u32 类型的。该命令输出结果一行对应 16 个字节，也就是 4 个 u32 。每列宽度可以通过 `-g` 选项控制，默认为 2，即 4 个字节。

下面输出结果中第一列为地址，后 4 列为二进制数据，之后部分，比如 `SFAR...` 为 ASCII 显示字符，在这里没有意义，可以忽略。

### RafsV5SuperBlock

先来看看 `RafsV5SuperBlock` 对应的数据。

#### 第一个 16 字节

```
00000000: 52414653 00000500 00002000 00100000  SFAR..... ......
```

前 16 个字节，对应 RafsV5SuperBlock 的以下属性：

- `s_magic: u32`： v5 magic number 是 `0x5241_4653`。
```rust
// rafs/src/metadata/layout/v5.rs
const RAFSV5_SUPER_MAGIC: u32 = 0x5241_4653;
```

- `s_fs_version: u32`： 文件系统版本，即 `500`。
```rust
// rafs/src/metadata/layout/mod.rs
pub const RAFS_SUPER_VERSION_V5: u32 = 0x500;
```

- `s_sb_size: u32`： super block 大小 0x2000，即 8192 字节。
- `s_block_size: u32`： 块大小，0x00100000 == 1048576 == 1024 * 1024。

#### 第二个 16 字节

```
00000010: 00000016 00000000 00000003 00000000  ................
```

对应两个属性：

- `s_flags: u64`： 来自 RafsSuperFlags ，这里值为 16，具体内容为 `COMPRESS_LZ4_BLOCK | DIGESTER_BLAKE3 | EXPLICIT_UID_GID`，如何算出来的如下所示：
```rust
// rafs/src/metadata/mod.rs
const COMPRESS_LZ4_BLOCK = 0x0000_0002;
const DIGESTER_BLAKE3 = 0x0000_0004;
const EXPLICIT_UID_GID = 0x0000_0010;
```

- `s_inodes_count: u64`： 这里值为3，即只有3个inode，一个根文件，还有两个普通文件。

#### 第三个 16 字节

```
00000020: 00002000 00000000 00002010 00000000  . ....... ......
```

对应两个属性：

- `s_inode_table_offset: u64`： inode table 所在位置的位移，这里为 0x2000 = 8192。注意这里虽然是 u64 ，但是好像 `00002000 00000000` 的顺序还是需要以 4 字节为单位，从右往左读，而每个 4 字节则是从左往右读。
- `s_prefetch_table_offset: u64`： prefetch table 的位移，从相对值来说，比上面的 inode table 的位置 多了 0x10，即 16 个字节。

#### 第四个 16 字节

```
00000030: 00002010 00000000 00000003 00000000  . ..............
```

对应三个属性：

- `s_blob_table_offset: u64`： blob table 对应的值为 0x2010 ，和 prefetch table 的 offset 值一样。大概是因为该 bootstrap 没有使用 prefetch 吧。
- `s_inode_table_entries: u32`： 3 个 entries 对应上面说的 3 个 inode 。
- `s_prefetch_table_entries: u32`： prefetch entries 数量为 0 。

#### 第5个 16 字节

```
00000040: 00000048 00000001 00002058 00000000  H.......X ......
```

对应三个属性：

- `s_blob_table_size: u32`： 0x48 == 72。
- `s_extended_blob_table_entries: u32`： 有 1 个条目，因为我们这个测试中只生成一个 blob 文件。
- `s_extended_blob_table_offset: u64`： 位移位置在 0x2058。

#### 第5个 16 字节及以后

```
00000050: 00000000 00000000 00000000 00000000  ................
```

`RafsV5SuperBlock` 结构体的最后一个属性如下：

- `s_reserved: [u8; RAFSV5_SUPERBLOCK_RESERVED_SIZE]`

我们从这个常量的定义可以看到：

```rust
// rafs/src/metadata/layout/v5.rs
pub(crate) const RAFSV5_SUPERBLOCK_SIZE: usize = 8192;
const RAFSV5_SUPERBLOCK_RESERVED_SIZE: usize = RAFSV5_SUPERBLOCK_SIZE - 80;
```

### Inode Table

整个 super block 占用 8192 字节，其中前面我们看到的几个属性，占用 80 个字节，其余部分为保留区域，以供扩展时使用。

`RafsV5SuperBlock` 中大部分都是预留的空位置，从下面一行可以看到，地址跳到了 0x2000，也是我们前面看到的 `s_inode_table_offset` 的值，这也意味着，这行开始的内容是 inode table 的内容。

在看上面的内容之前，我们先看看 inode table 的结构：

```rust
// rafs/src/metadata/layout/v5.rs
pub struct RafsV5InodeTable {
    /// Inode offset array.
    pub data: Vec<u32>,
}
```

注意这里 inode table 是一个 vector，里面存的不是 inode 对象，而是 inode 对应的 offset，而 vector 的索引就是 inode 对应的数值，每个 offset 都是 32 位类型，占用 4 个字节。而且索引的值为 inode -1 ，这样 root inode 就保存在 0 的位置上，一点都不浪费。

```
00002000: 00000413 00000424 00000435 00000000  ....$...5.......
```

我们看到了 3 个 offset，分别为 413、424 和 432，每个 offset 之间间隔 11 。这个 offset 是 inode table 元素对应在 bootstrap 文件中的绝对位置的偏移量，比如 0x413 = 1043，而这个数据结构保存的位置都是 8 字节对齐的，所以这个值在保存的时候，是右位移 3 位的，取出来的时候再左位移 3 位，所以真正的位移值为 1043 * 8 = 8344，也就是 16 进制的 0x2098，在下面的分析中我们还会看到 inode table 的具体内容。

同理 0x424 对应的偏移为 0x2120，0x432 对应的偏移量是 0x2190。

**注意**： rafs v5 存储都有对齐，为 8 字节。由于在这里的测试中只有 3 个文件，所以存储需要 3 个 u32，但是由于存在 8 字节对齐的需求，这里 3 个 u32 是 12 个字节，所以需要补齐 4 个字节，所以我们看到 0x2000 那一行最后补齐的全零的 u32。

```rust
pub(crate) const RAFSV5_ALIGNMENT: usize = 8;
```

### Blob table

从 0x2010 开始存储的是 blob table 内容。还是先来看看 blob table 的定义：

```rust
// rafs/src/metadata/layout/v5.rs
#[derive(Clone, Debug, Default)]
pub struct RafsV5BlobTable {
    /// Base blob information array.
    pub entries: Vec<Arc<BlobInfo>>,
    /// Extended blob information array.
    pub extended: RafsV5ExtBlobTable,
}
```

注意，BlobInfo 结构是在 storage crate 中定义的。

```rust
// storage/src/device.rs

/// Configuration information for a metadata/data blob object.
///
/// The `BlobInfo` structure provides information for the storage subsystem to manage a blob file
/// and serve blob IO requests for clients.
#[derive(Clone, Debug, Default)]
pub struct BlobInfo {
    /// The index of blob in RAFS blob table.
    blob_index: u32,
    /// A sha256 hex string generally.
    blob_id: String,

    /// 此处省略其余属性
}
```

前面我们已经看到，s_blob_table_offset 对应的值为 0x2010 ，s_blob_table_size 的值为 0x48 = 72 字节，也就是存储位置在 [2010 - 2058)，2058 也正是 s_extended_blob_table_offset 的值。这部分数据如下： 

```
00002010: 00000000 00000000 31343261 65373762  ........a241b77e
00002020: 38333362 32373532 63623763 38336231  b3382572c7bc1b38
00002030: 38623561 36393139 36326366 62343062  a5b89196fc26b04b
00002040: 37363666 34313962 63653062 33313137  f667b914b0ec7113
00002050: 37343061 32623835 00000001 00000000  a04758b2........
00002050: 37343061 32623835   00000001 00000000  a04758b2........
```

注意上面 2050 这一行我又拷贝了一遍，并在 2058 前添加了 2 个空格来方便识别位置。

这里我们 blob 的 id 是 `a241b77eb3382572c7bc1b38a5b89196fc26b04bf667b914b0ec7113a04758b2`，可以和上面输出最右侧 ASCII 部分内容对照。

而且要注意的是上面的输出左右对照稍微不太直观，虽然列是从左到右，但是一列之中是从右到左的顺序。比如 `31343261` ，对应的内容实际是 `61323431`，即 `a241`。

虽然上面看到的 blob table 的定义很复杂、属性很多，但是我们的测试足够简单。RafsV5BlobTable 存储到磁盘后，最开始的内容就是 BlobInfo 结构体。

但是在 RafsV5BlobTable 的 store 方法（用于序列化到磁盘的 RafsStore trait 所需）中，我们可以看到，对于每一个 blob info 对象，都会在前面写两个 readahead 属性，这两个属性供占用 8 个字节，然后才是 blob id 。所以我们可以在上面 bootstrap 内容中，在 8 个自己的全零（默认值即是 0 ）后面，才是 blob id。

```rust
w.write_all(&u32::to_le_bytes(entry.readahead_offset() as u32))?;
w.write_all(&u32::to_le_bytes(entry.readahead_size() as u32))?;
w.write_all(entry.blob_id().as_bytes())?;
```

尽管 BlobInfo 对象属性很多，但是持久化到磁盘的内容却不多，主要就上面这 3 个，外加一些对齐的字节。

下面从 0x2058 开始是 RafsV5ExtBlobTable 的内容。其定义如下：

```rust

/// Rafs v5 on disk extended blob information table.
#[derive(Clone, Debug, Default)]
pub struct RafsV5ExtBlobTable {
    /// The vector index means blob index, every entry represents
    /// extended information of a blob.
    pub entries: Vec<Arc<RafsV5ExtBlobEntry>>,
}

/// Rafs v5 extended blob information on disk metadata.
///
/// RafsV5ExtDBlobEntry is appended to the tail of bootstrap,
/// can be used as an extended table for the original blob table.
// This disk structure is well defined and rafs aligned.
#[repr(C)]
#[derive(Clone)]
pub struct RafsV5ExtBlobEntry {
    /// Number of chunks in a blob file.
    pub chunk_count: u32,
    pub reserved1: [u8; 4],     //   --  8 Bytes
    pub uncompressed_size: u64, // -- 16 Bytes
    pub compressed_size: u64,   // -- 24 Bytes
    pub reserved2: [u8; RAFSV5_EXT_BLOB_RESERVED_SIZE],
}
```

其中 `RAFSV5_EXT_BLOB_RESERVED_SIZE` 的值为 40。

对齐进行持久化的方法如下：

```rust
w.write_all(&u32::to_le_bytes(entry.chunk_count))?;
w.write_all(&entry.reserved1)?;
w.write_all(&u64::to_le_bytes(entry.uncompressed_size))?;
w.write_all(&u64::to_le_bytes(entry.compressed_size))?;
w.write_all(&entry.reserved2)?;
```

看完代码再来看一下存储的数据内容。

```
00002050: 37343061 32623835 00000001 00000000  a04758b2........
00002060: 00000040 00000000 00000035 00000000  @.......5.......
```

注意 0x2058 是从上面第 9 个字节开始的，这里 chunk_count 的值为 0x00000001，即只有一个 chunk。然后 uncompressed_size 属性占用 8 个字节，在上面两行中实际是跨行了，包括第一行的后四个和第二行的前四个字节。其值为 0x40，即 64 字节。同理，compressed_size 的值为 0x35，即 53 字节。这也和我们在文件系统上看到的是一样的：

```bash
-rw-r--r-- 1 vagrant vagrant   53 Apr 26 06:55 a241b77eb3382572c7bc1b38a5b89196fc26b04bf667b914b0ec7113a04758b2
```

0x2060 行最后的 4 个字节为保留字节。

Blob table 之后写入是 inode 信息，这些信息是通过 RafsV5InodeWrapper 结构表示的。Inode 信息由 3 部分组成：

- Inode 结构体的数据
- xattrs
- Chunk info

我们还是先来看看一些关键数据结构的定义。

```rust
// rafs/src/metadata/layout/v5.rs
pub struct RafsV5InodeWrapper<'a> {
    pub name: &'a OsStr,
    pub symlink: Option<&'a OsStr>,
    pub inode: &'a RafsV5Inode,
}

// 删除了该方法的一些内容，主要保留了核心写入到磁盘的数据
impl<'a> RafsStore for RafsV5InodeWrapper<'a> {
    fn store(&self, w: &mut dyn RafsIoWrite) -> Result<usize> {
        let mut size: usize = 0;

        // 1. 写入 RafsV5Inode 内容，128 字节
        let inode_data = self.inode.as_ref();
        w.write_all(inode_data)?;

        // 2. 写入文件名，可变长度
        let name = self.name.as_bytes();
        w.write_all(name)?;
    }
}

pub struct RafsV5Inode {
    /// sha256(sha256(chunk) + ...), [char; RAFS_SHA256_LENGTH]
    pub i_digest: RafsDigest, // 32
    /// parent inode number
    pub i_parent: u64,
    /// from fs stat()
    pub i_ino: u64,
    pub i_uid: u32,
    pub i_gid: u32,
    pub i_projid: u32,
    pub i_mode: u32, // 64
    pub i_size: u64,
    pub i_blocks: u64,
    pub i_flags: RafsV5InodeFlags,
    pub i_nlink: u32,
    /// for dir, child start index
    pub i_child_index: u32, // 96
    /// for dir, means child count.
    /// for regular file, means chunk info count.
    pub i_child_count: u32,
    /// file name size, [char; i_name_size]
    pub i_name_size: u16,
    /// symlink path size, [char; i_symlink_size]
    pub i_symlink_size: u16, // 104
    // inode device block number, ignored for non-special files
    pub i_rdev: u32,
    // for alignment reason, we put nsec first
    pub i_mtime_nsec: u32,
    pub i_mtime: u64,        // 120
    pub i_reserved: [u8; 8], // 128
}
```

我们继续对照 dump 出来的 bootstrap 数据来看一下写入的具体内容。

```
00002070: 00000000 00000000 00000000 00000000  ................
00002080: 00000000 00000000 00000000 00000000  ................
00002090: 00000000 00000000 afbe1b2a 8b68b09e  ........*.....h.
000020a0: ac7a3553 ec9df26a 5d07bafa 02097de0  S5z.j......].}..
000020b0: 4c85264b 5749c4a7 00000000 00000000  K&.L..IW........
000020c0: 00000001 00000000 000003e8 000003e8  ................
000020d0: 00000000 000041ed 00000080 00000000  .....A..........
000020e0: 00000001 00000000 00000000 00000000  ................
000020f0: 00000002 00000002 00000002 00000001  ................
00002100: 00000000 00000000 00000000 00000000  ................
00002110: 00000000 00000000 0000002f 00000000  ......../.......
```

首先我们肉眼可见的 128 长度的 sha256 摘要，所以很容易定位到 RafsV5Inode 结构的内容。之后的 8 字节表示 parent inode，这里为 0。

然后到着看找到 2110 行的 `2f`，这就是 ASCII 的 / ，也就是根目录。再倒着往回数 26 个字节，到了 i_name_size 属性，这里值为 1，即 `/` 长度为 1，再往上一个属性，即 i_child_count ，这里值为 2。其余属性这里就不再详细介绍。

```
00002120: b94913af a6a1f9f5 ea4d40a0 49c9dc36  ..I......@M.6..I
00002130: c925cb9b b712c1ad ca939acc 62321fe4  ..%...........2b
00002140: 00000001 00000000 00000002 00000000  ................
00002150: 000003e8 000003e8 00000000 000081a4  ................
00002160: 00000000 00000000 00000000 00000000  ................
00002170: 00000000 00000000 00000001 00000000  ................
00002180: 00000000 00000003 00000000 00000000  ................
00002190: 626767b2 00000000 00000000 00000000  .ggb............
000021a0: 00616161 00000000 b232f6e2 e21610c0  aaa.......2.....
```

同样下一个文件名为 `aaa`，也可以从 `00616161` 看到。

```
000021a0: 00616161 00000000 b232f6e2 e21610c0  aaa.......2.....
000021b0: efe31e11 2532c9d6 2f8e943d e7082bfe  ......2%=../.+..
000021c0: 81da0118 d1192211 00000001 00000000  ....."..........
000021d0: 00000003 00000000 000003e8 000003e8  ................
000021e0: 00000000 000081a4 00000040 00000000  ........@.......
000021f0: 00000001 00000000 00000000 00000000  ................
00002200: 00000001 00000000 00000001 00000003  ................
00002210: 00000000 00000000 62679767 00000000  ........g.gb....
00002220: 00000000 00000000 00626262 00000000  ........bbb.....
```

然后是最后一个文件 `bbb`，也可以从 `00626262` 看到。它的父文件夹为 1，文件大小为 0x40，这里应该是压缩后的文件大小了。


最后是 chunk 信息。我们来看看它的结构：

```rust
// rafs/src/metadata/layout/v5.rs
pub struct RafsV5ChunkInfo {
    /// sha256(chunk), [char; RAFS_SHA256_LENGTH]
    pub block_id: RafsDigest, // 32
    /// blob index.
    pub blob_index: u32,
    /// chunk flags
    pub flags: BlobChunkFlags, // 40
    /// compressed size in blob
    pub compress_size: u32,
    /// uncompressed size in blob
    pub uncompress_size: u32, // 48
    /// compressed offset in blob
    pub compress_offset: u64, // 56
    /// uncompressed offset in blob
    pub uncompress_offset: u64, // 64
    /// offset in file
    pub file_offset: u64, // 72
    /// chunk index, it's allocated sequentially and starting from 0 for one blob.
    pub index: u32,
    /// reserved
    pub reserved: u32, //80
}
```

可以看出，它的长度是 80 个字节，对应文件的最后这部分：

```
00002230: ec5944de 690964ef 8274f1bf 7cf30f7c  .DY..d.i..t.|..|
00002240: 62fc5b93 d8d8c7a0 3272f74b b91db007  .[.b....K.r2....
00002250: 00000000 00000001 00000035 00000040  ........5...@...
00002260: 00000000 00000000 00000000 00000000  ................
00002270: 00000000 00000000 00000000 00000000  ................
```

上面只是简单分析了下 bootstrap 文件的内容，至于 blob 数据文件，则留待以后有时间再看了。