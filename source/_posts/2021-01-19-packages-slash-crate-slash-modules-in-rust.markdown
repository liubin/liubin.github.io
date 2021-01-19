---
layout: post
title: "Rust 学习笔记：package/crate/module"
date: 2021-01-19 20:45:37 +0800
comments: true
categories: 
---

cargo new 会生成项目的雏形，提供了src/main.rs和src/lib.rs文件，但是随着项目的增长，代码的量也会变大，靠一个文件维护一大堆代码，肯定是不合适的。这时候一般都会按“模块”来拆分文件，rust也不例外。

这里学习一下rust中代码的组织方式，主要涉及到以下几个概念：

- package：Cargo中的概念，管理crate
- crate：模块的集合，编译单位，有lib和bin两种，即供别人调用，或者是一个可执行文件
- module：用于在crate内组织代码
- workspace：项目复杂时，管理多个package

## package

cargo new 命令会创建一个新项目，也是一个package，里面有一个Cargo.toml文件，用于定义package、所需外部依赖，以及如何编译crate等。

## crate

rust里有两种crate，lib类型和bin类型，并且默认以文件名为标准按以下规则处理crate：

- src/main.rs：表示该crate是一个bin类型的crate
- src/lib.rs：表示该crate是一个lib类型的crate

src/main.rs和src/lib.rs都是crate的根，也就是crate引用、rustc编译的入口。

此外，一个package中的crate还有如下约束：

1. 多个bin类型的crate
1. 0个或1个lib类型的crate

其中，1和2并不互斥，也就是说一个项目下可以有1个lib和多个bin类型的crate，即一个package还以编译出多个可执行文件。

只是如果有多个bin类型的crate，一个src/main.rs就不行了，就得放到 src/bin 下面，每个crate一个文件，换句话说，每个文件都是一个不同的crate。

## mod

代码多了可以对代码以mod（文件/文件夹）为单位进行拆分，而不必把所有代码都写在src/lib.rs或者src/main.rs里。

以lib类型的crate为例，该crate的入口在src/lib.rs，也是crate的根。在 src/lib.rs 里定义模块很简单：

```rust
mod mymod {
    fn test() {
        println!("test");
    }
}
```

而实际项目中，我们都不可能只有一个lib.rs文件，而是会将代码按功能等拆分为多个模块。

### 模块拆分
一般来说，一个文件都会被视为一个mod，而且mod可以嵌套定义。嵌套定义的mod既可以写在同一个文件里，也可以通过文件夹的形式来实现。

具体我们来看几个例子。

假设当前项目文件结构如下：

```
src
├── lib.rs
├── mod_a
│   ├── mod.rs
│   └── mod_b.rs
└── mod_c.rs
```

这里显示定义了3个mod：mod_a、mod_b和mod_c，其中mod_a为文件夹形式，而mod b 和mod c都有对应的文件。其中mod_b是mod_a的子模块。

我们来看一下各个模块是怎么声明的，以及应该如何引用。

首先来看一下crate的根，也就是入口lib.rs：

```rust
pub mod mod_a;
mod mod_c;
```

这里声明了两个mod，如果需要在crate外部访问，可以在mod前面加上pub关键字。注意这里不需要声明mod_a的子模块mod_c，这个需要由mod_a来声明。

再来看一下这两个mod。先看mod_a，这是一个文件夹形式存在的mod，按cargo规定，这时候需要在该文件夹下有一个名为mod.rs的文件定义该mod下的内容。该文件内容如下：

```rust
// src/mod_a/mod.rs
pub mod mod_b;
```

可以看到，这个文件和lib.rs类似，都可以声明mod。该文件声明的mod_b的代码则保存为mod_b.rs：

```rust
// src/mod_a/mod_b.rs
use super::super::mod_c;

pub fn test() {
    println!("i'm mod_b");
}

fn call_mod_c() {
    mod_c::test();
}
```

再来看一下mod_c的代码：

```rust
// src/mod_c.rs
use crate::mod_a::mod_b;

pub fn test() {
    mod_b::test();
    println!("i', mod_c");
}

```

除了如何定义mod，还需要注意的是如何引用其他mod的定义。这里在mod_c中，要想使用mod_b，可以使用 **use crate::mod_a::mod_b** 这种绝对路径形式。

而在mod_b中使用mod_c的时候，使用了 **use super::super::mod_c** 这种先对路径的形式。

### 添加main.rs

最后在上面代码的基础上添加一下main.rs，看看如何作为外部crate使用上面的mod_a。

```rust
// src/main.rs
use testlib::mod_a::mod_b;

fn main() {
    println!("main");
    mod_b::test();
}
```

这里唯一想要提醒的就是lib的引用方法，不能使用crate开头的绝对路径或者相对路径引用方式，必须使用该crate的名称（也就是Cargo.toml里的名称，本例为testlib）来引用。因为main和lib分别属于不同的crate。

假如将上面的testlib改为crate，编译器会报如下错误：

{% img center-image /images/2021/01/rust-compile-error.png '编译错误' %}

很多时候编译器都是我们最好的老师。

### pub修饰符
#### 结构体和枚举

要想访问其他mod里的结构体，需要将结构体声明为pub，但是这也只能访问到结构体而已，如果要想操作里面的字段，可以有两种方式：

- 提供pub的方法修改字段
- 将需要操作的字段直接修改为pub类型

可能前者更“面向对象”一些。

而枚举类型的话只需要在枚举名前面加上pub即可，不需要对其中的variant进行设置。

## use语句

讲了这么多基础概念，下面看一下如何使用。

在crate和模块了可能定义了函数、结构体等，要想在其他模块或crate使用，需要将其引入到当前scope中，类似java的import的功能，rust里需要使用use。

如何表示要被引用的对象（rust中称为item），rust里称之为path，可以理解为我们在操作系统里使用文件一下。

rust中path有两种形式，也跟文件系统一样，绝对路径和相对路径：

- 绝对路径始于crate的根（src/main.rs或src/lib.rs），可以使用crate名或者crate这个字面值表示
- 相对路径可以使用当前模块名、当前模块中可以使用的对象，super和self等。

path中的层级使用两个冒号，类似文件系统中的斜线。

假设有如下代码（来自trpl）：

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // Absolute path
    crate::front_of_house::hosting::add_to_waitlist();

    // Relative path
    front_of_house::hosting::add_to_waitlist();
}
```


上面的第9行就是绝对路径形式的引用，而第12行就是相对路径的引用，这里，front_of_house处于crate的根之下，而不是位于其他子模块之下。

有一些限制需要知道：

- 在父模块中不能使用子模块中的private项目
- 子模块可以使用父模块中的所有item

注意 front_of_house 模块虽然不是pub的，但是eat_at_restaurant却可以使用，因为他们在同一模块下，这不需要pub就可以使用，否则所有item都只能变成pub才能使用了。但是hosting模块和add_to_waitlist方法必须为pub类型的，否则就不能从他们的父模块中的项目中使用了。

下面是一个使用了super的例子：

```rust
fn serve_order() {}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```

fix_incorrect_order方法属于back_of_house模块，要调用的serve_order和back_of_house同级，因此在back_of_house模块中的函数里，可以使用super::serve_order，访问到该模块同级的serve_order方法。

如果use后面的路径具有具有共同的父路径，可以使用简化的模式。比如 ：

```rust
use std::io;
use std::cmp::Ordering;
```

可以简化为：

```rust
use std::{cmp::Ordering, io};
```

如果同时use的mod之间有父子关系，也可以像上面那样简化，使用self代表父mod。比如：

```rust
use std::io;
use std::io::Write;
```

可以简化为：

```rust
use std::io::{self, Write};
```

如果想将某一路径下的所有public的item都引入到当前scope中，可以使用`*`。

```rust
use std::collections::*;
```

一般业务代码文件内的单元测试中常用：

```rust
#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn it_works() {
    }
}
```

这样在单测mod中，可以使用父mod中的所有item。


### 引用层级

对比两段代码：

```rust
# case 1
use crate::front_of_house::hosting;
hosting::add_to_waitlist();

# case 2
use crate::front_of_house::hosting::add_to_waitlist;
add_to_waitlist();
```

这两种方法结果一样，但是阅读起来给人的感觉是不一样的。一般来说推荐前者，因为这样可以明确的知道使用的方法是外部的hosting模块的方法，后者的话则不知道该方法是use进来的，还是本模块定义的。

### 名称冲突
有的时候可能从不同的crate或者mod引入了同名的item，这时候最简单的方式是使用 as 关键字进行重命名。

```rust

use std::fmt;
use std::io;

fn function1() -> fmt::Result {
}

fn function2() -> io::Result<()> {
}


use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
}

fn function2() -> IoResult<()> {
}
```


### re-exporting 再导出

当使用use关键字将外部item导入到当前scope之后，这个item在当前scope是private的，如果使用 pub use 的话，还能让使用当前mod的第三者，使用在该mod中引入的item。

该机制称为 re-exporting 。

## workspace

workspace用于管理多个相关的package，不同的package有各自的Cargo.toml，但是整个workspace共享一个Cargo.lock，也只有一个target目录（编译输出）。

虽然workspace内的项目共享一个Cargo.lock，但是他们之间默认不互相依赖，需要显示添加它们之间的依赖关系。而且在一个项目中添加的依赖，在其他项目中如果想使用，还需要再次声明依赖才行。

不过据我观察workspace功能没有什么特别强大之处，不使用该功能也可以同时管理几个Cargo项目，因此这里就不再深入介绍了。
