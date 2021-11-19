---
layout: post
title: "Array/Slice/Vector in Rust"
date: 2021-11-19 15:35:15 +0800
comments: true
categories: 
---

不管哪种编程语言，最常用的数据类型不外乎数值、字符串以及数组。这里数组是一种泛称，一般指能存放多个元素的集合，当然这里的集合也不是严格的数学定义。

## Array

先来看数组。

一个 array 是一组相同类型的数据集合，这些数据位于连续的内存块中，且保存在栈上而不是堆上。

以下都是数组的基本使用方式。


```rust
fn main() {
    let a = [1, 2, 3, 4];
    assert_eq!(1, a[0]);

    let a: [i32; 4] = [1, 2, 3, 4];
    assert_eq!(4, a[3]);

    let a = [0; 4];
    assert_eq!(0, a[2]);

    let mut a = [1, 2, 3];
    assert_eq!(1, a[0]);

    a[0] = 11;
    assert_eq!(11, a[0]);
}

```

主要有 3 种方法创建数组：

- `[V]`：直接使用元素值，不指定数组类型，比如 `let a = [1, 2, 3, 4]`
- `[T; N]`：声明一个类型为 T 长度为 N 的数组，比如 `let a: [i32; 4]`
- `[V; N]`：声明一个每个元素值为 V，长度为 N 的数组，比如 `let a: [0; 4]` 。当然数组的类型为 V 的类型 T。


如果我们声明中使用了可变关键字 mut ，则表示该数组的元素值可以被修改：

```rust
let mut a = [1, 2, 3];
a[0] = 11;
```

Rust 中数组最大的特点是大小不可变，这对于一些编程语言用户来说能难以理解，也就是说虽然我们能修改数组中某个元素的值，但是不能往里添加元素，或者删除元素。



总结一下就是数组在编译时分配大小，保存在栈上，数组长度不可变，虽然我们能改变数组中某个元素的值。


## Vector

我们大多数情况下都希望“数组”元素的个数是可变的，这时候可以使用 Vector 类型。

Vector 的使用方式如下：



```rust
fn main() {
    let v = vec![1, 2, 3];
    assert_eq!(1, v[0]);

    let v = vec![4; 3];
    assert_eq!(4, v[0]);

    let mut v: Vec<i32> = Vec::new();
    v.push(5);
    assert_eq!(5, v[0]);
}
```


Vector 主要有两种初始化方式：

- 使用 `vec!` 宏
- 使用 `Vec::new()` 方法

Vector在内存中由三部分组成：
- 指向堆地址的指针，因为分配在堆上，所以才能动态修改大小
- 现有元素个数
- Vector 的容量（capacity），即一共可以保存多少个元素。

像其他语言一样， Rust 在创建 Vector 的时候，也可以预留一定数量的存储空间，以防止在元素增加时因元素移动、拷贝导致的开销。

```rust
let mut v: Vec<i32> = Vec::with_capacity(3);
assert_eq!(0, v.len());
assert_eq!(3, v.capacity());

v.push(1);
assert_eq!(1, v.len());
assert_eq!(3, v.capacity());
v.push(2);
v.push(3);
assert_eq!(3, v.len());
assert_eq!(3, v.capacity());

v.push(4);
assert_eq!(4, v.len());
assert_eq!(6, v.capacity());
```

我们可以使用 `Vec::with_capacity()` 来预分配指定大小的存储空间，然后如果实际元素个数超过这个值时，Vector 的容量会扩展，我们看到当元素个数超过预分配的 3 之后，capacity 涨到了 6。

从上面我们也可以看到，Array 最大的一个限制是它的固定大小。与此相对，Vector 的特点是：

- 分配在堆上
- 可以在运行时动态添加或删除元素

## Slice

我们再来看一下 Slice ，一般语言翻译为切片。Slice 一般是指向 Array 或 Vector 的位置，一般用 `&[T]` 表示。

我们可以通过使用指向 Array 或 Vector 的 range 来创建 Slice 类型的变量。Slice 在使用的时候和 Array 非常像。

```rust
fn main() {
    let a = [1, 2, 3, 4, 5];
    let v = vec![1, 2, 3, 4, 5];

    let sa: &[i32] = &a;
    let sv: &[i32] = &v;

    assert_eq!([1, 2, 3, 4, 5], sa);
    assert_eq!([1, 2, 3, 4, 5], sv);

    let sa: &[i32] = &a[1..3];
    assert_eq!([2, 3], sa);

    let sa: &[i32] = &a[1..=3];
    assert_eq!([2, 3, 4], sa);
}

```

Slice 在实现上被称作 fat pointer。fat pointer 在内存中保存了两个值：
- Slice 指向的位置
- Slice 包含的元素数量


Rust 中，`&[T]`、`&mut [T]`,、`Box<[T]>` 等类型在内存中占有 16 个字节，其中前 8 个字节为指针位置，后 8 个字节为 Slice 包含的元素个数，即长度。

```
 0-7 | 8-15
-----|------
 ptr | len
```

而 `Vec<T>` 在内存中结构如下：

```
 0-7 | 8-15 | 16-23
-----|------|-------
 ptr | cap  |  len
```

在内存中 `Vec<T>` 占用 24 个字节，和 Slice 相比，多了 8 个字节的空间来存储 capacity 属性。

有了静态分配大小的 Array，以及可以动态增减元素的 Vector，为什么还有一个叫做 Slice 的东西呢？按照官方文档的说明，Slice是一个指向底层 Array 或 Vector 的视图，可以实现安全、高效的数据访问而不会发生内存拷贝。一般来说我们都不会直接创建 Slice ，而是从已有的 Array 或 Vector 创建 Slice。


Slice 虽然是一个视图（view），但是也可以通过 `&mut [T]` 它来修改底层元素：

```rust
let mut v = vec![1, 2, 3, 4, 5];
let sv: &mut [i32] = &mut v;
sv[1] = 9;
assert_eq!(9, sv[1]);
```

## String and `&str`

最后稍微来介绍一下 String 和 &str 的关系，这有点像 Vector 和 Slice 的对应关系关系。在内存中，&str也包括指向实际数据位置的指针和长度属性。

字符串 Slice 也可以看做是 8 bit 整数 Slice 重新定义的类型，两者的对应关系如下：

```
Byte slice | 字符串 slice
------------|-------------
 &[u8]      | &str
 Vec<u8>    | String
```



