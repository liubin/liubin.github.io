---
layout: post
title: "async/await 内部是怎么实现的"
date: 2021-04-15 16:50:22 +0800
comments: true
categories: 
---

同事问 Rust aysnc/await 是怎么实现的呢，在 await 的地方停住，之后又在继续的时候继续恢复（当前线程/coroutine）的执行，也是用了 yield/generator 这样的东西？

简单的试了下，猜测大概是这样吧。

如下代码：

```rust
async fn say_world() {
    println!("hello world");
}

#[tokio::main]
async fn main() {
    let op = say_world();

    op.await;
}
```

使用 nightly 的 `rustc` “编译”：

```
$ cargo rustc -- -Z unpretty=hir
```

下面是输出结果，这里只显示了 `main()` 函数相关处理后的代码（修改过格式后）：

```
tokio::runtime::Builder::new_multi_thread().enable_all().build().unwrap().block_on(#[lang = "from_generator"](|mut _task_context|
{
    let op = say_world();
    match op
        {
        mut pinned => loop {
            match unsafe
                  {
                      #[lang = "poll"](#[lang = "new_unchecked"](&mut pinned),
                                       #[lang = "get_context"](_task_context))
                  }
                {
                    #[lang = "Ready"] {
                    0: result
                    } =>
                    break result,
                    #[lang = "Pending"] { } =>
                    {
                    }
                }

            _task_context = (yield());
        },
    };
}))
```

抛去那么多 attribute ，大概流程就是不挺的 loop ，查看 Future（这里的 `op`） 是否 ready。如果已经是 ready 的状态，那么就会对该结果进行处理，然后退出；否则（Pending的状态）就继续等待，让 runtime 调度其他 task 。

Future 在 tokio 里就“是”一个 task（确切说是 future.await？），tokio runtime 负责调度 task ，task 有些像 goroutine，不过 Rust 本身不自带 runtime 的实现。

计划再确认下 tokio 的实现，看看它到底是怎么做的。
