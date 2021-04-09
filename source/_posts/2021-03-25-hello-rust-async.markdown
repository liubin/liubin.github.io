---
layout: post
title: "Hello Rust async/await"
date: 2021-03-25 19:26:43 +0800
comments: true
categories: 
---

Rust 对 async/await 的支持越来越成熟了，在一些场景下相对于线程等模型能显著提高效率。

这里我们来简单了解下怎么在 Rust 最快速的入手异步编程。

## Hello world async/await

在 Rust 中，异步编程都抽象为 Future trait，类似 JavaScript 中的 Promise 。在最近的 Rust 中，直接使用 async 关键字即可创建 Future 对象。

async 关键字可以用于创建如下类型的 Future：

- 定义函数：`async fn`
- 定义 block： `async {}`

Future 不会立即执行，要想执行 Future 定义的函数，需要：

- 使用 `await`
- 或者在异步运行时中为该 Future 创建 task

创建异步任务，可以选择如下方式：


- 使用 `block_on`
- 使用 `spawn`


## 使用 async/await 关键字

这里我们以 tokio 为例来看一些简单的入门示例，来加深一下对这几个概念的理解。

```rust
async fn world() -> String {
    "world".to_string()
}

async fn hello() -> String {
	// 使用 .await 关键字调用 world() 函数
    let w = world().await;
    format!("hello {} async from function", w)
}

fn main() {
	// 创建运行时
    let rt = tokio::runtime::Runtime::new().unwrap();
    // 使用 block_on 调用 async 函数
    let msg = rt.block_on(hello());
    println!("{}", msg);

    // 使用 block_on 调用 async block
    let _ = rt.block_on(async {
        println!("hello world async from block");
    });
}
```

在这个例子中，hello 和 world 函数都使用了 async 关键字，表示该函数要以异步的方式执行。两个函数的返回值本来为 String，但是加了 async 关键字后，这两个函数的最终的签名会在内部表示为 `fn hello() -> impl Future<Output=String>`。即返回值是一个 Future 类型，这个 Future 执行后，会返回 String 类型的结果。

这里我们使用了两种方法执行 Future。

在 hello 函数中，使用了 `world().await` 来调用 world 函数，并等待该函数返回，其结果不是 Future 类型，而是 Future 关联的 Output 类型，在这里即 String。


除了直接使用 await 关键字，我们还使用了 `tokio::runtime::Runtime::new()` 创建了 tokio 运行时，并在其中运行我们的 Future ，即 `rt.block_on(hello())` 和 `rt.block_on(async {})` 这两处。

对于 async block，也可以直接调用 await：


```rust
async {
    println!("hello world async");
}.await;
```



其实，tokio 提供了一个非常方便的注解（或称属性），方便我们在 main 函数中执行 Future 任务。


```rust
#[tokio::main]
async fn main() {
    let msg = hello().await;
    println!("{}", msg);

    async {
        println!("hello world async from block");
    }
    .await;
}
```

只需要在 main 上面加上 `#[tokio::main]` ，前面加上 async 关键字，即可在其内部直接执行 await 方法，而不必使用 block_on 或者 spawn 方法。


**Tip:** `async` 关键字可以创建一个 Future ，与之相对，`.await` 则会销毁（解构）这个 Future. 因此，我们也可以说这两个关键字互相消解，`async { foo.await }` 就相当于 `foo`。

## 使用 spawn

前面的例子直接执行了 Future 任务，我们也可以使用 spawn 来创建 Future 的任务，然后让任务并行执行，并获取任务的执行结果。


spawn 会启动一个异步任务，并返回 JoinHandle 类型的结果。这个任务虽然启动，但是 spawn 不保证（等待）它会正常执行完成。考虑如下代码：

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        sleep(Duration::from_millis(100)).await;
        println!("hard work finished");
    });
    println!("mission started");
}
```

执行上面的代码，我们只能看到 `mission started` 打印出来，而不会看到异步任务的输出，这是因为在异步任务输出语句执行之前，main 函数就结束了，进程将会退出，异步任务中的打印语句将不会有机会执行。

这时候，我们需要使用 JoinHandle 来确保该任务执行完成。

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let jh = tokio::spawn(async {
        sleep(Duration::from_millis(100)).await;
        println!("hard work finished");
    });
    println!("mission started");

    let _ = jh.await.unwrap();
}
```

这里我们只需要拿到 spawn 的 JoinHandle ，并使用 await，即可以等待该任务结束，从而确保在所有工作完成后，再退出 main 函数。

JoinHandle 也可以用于获取异步任务的返回值，这里我们看一个 [官方文档](https://docs.rs/tokio/1.4.0/tokio/task/struct.JoinHandle.html) 的示例：

```rust
use tokio::task;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
    let join_handle: task::JoinHandle<Result<i32, io::Error>> = tokio::spawn(async {
        Ok(5 + 3)
    });

    let result = join_handle.await??;
    assert_eq!(result, 8);
    Ok(())
}
```

我们也可以用类似 golang 中的 chan 的机制来实现不同的异步任务之间的通信：

```rust
use tokio::sync::oneshot;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async {
        sleep(Duration::from_millis(100)).await;
        println!("hard work finished");
        tx.send("ping".to_string()).unwrap();
    });

    println!("mission started");
    let _ = rx.await.unwrap();
    println!("mission completed");
}
```

上面代码的输出结果为：

```
mission started
hard work finished
mission completed
```

可以看到，在 rx.await 的时候，main 函数会等待，直到异步任务结束之后，通过 `tx.send` 发送消息给了这个 chan ，main 函数才继续执行下面的步骤，打印了 "mission completed" 之后退出。

## 等待多个异步任务

有很多时候，我们可能会在开始启动多个异步任务，并等待所有异步任务执行完成。

tolio 提供了 `tokio::join!` 宏来实现该功能。

```
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let f1 = async {
        sleep(Duration::from_millis(100)).await;
        1
    };
    let f2 = async {
        sleep(Duration::from_millis(50)).await;
        "hello".to_string()
    };

    let (r1, r2): (i32, String) = tokio::join!(f1, f2);
    println!("r1:{}\nr2:{}", r1, r2);

    // 简单等待前面的异步任务结束
    sleep(Duration::from_millis(1000)).await;
}

```

要注意，`tokio::join!` 只有所有的异步任务都结束的时候才会返回。

如果想同时启动几个任务，只需要一个返回就继续进行后续处理的话，可以使用 select 宏。

```rust
use tokio::time::{sleep, Duration};

async fn test_select(t1: u64, t2: u64, timeout: u64) {
    let f1 = async {
        sleep(Duration::from_millis(t1)).await;
        1
    };
    let f2 = async {
        sleep(Duration::from_millis(t2)).await;
        "hello".to_string()
    };

    let timeout = sleep(Duration::from_millis(timeout));

    tokio::select! {
        _ = timeout => {
            println!("got timeout!");
        }
        v = f1 => {
            println!("got r1: {}", v);
        }
        v = f2 => {
            println!("got r2: {}", v);
        }
    }
}

#[tokio::main]
async fn main() {
    // got first task result
    test_select(100, 200, 500).await;
    // got second task result
    test_select(200, 100, 500).await;
    // timeout
    test_select(200, 100, 50).await;
}

```

上面的程序执行结果如下：

```
got r1: 1
got r2: hello
got timeout!
```

首先，我们在 test_select 方法中，定义了另外两个异步任务，分别返回整数型和字符串类型的值，并且分别设置了不同的 sleep 时间。我们分 3 次调用了这个方法：

- test_select(100, 200, 500).await;
- test_select(200, 100, 500).await;
- test_select(200, 100, 50).await;

其中前两个参数是两个异步任务的 sleep 时间，第三个参数是超时时间。从这三次调用所使用的参数来看，第三次超时时间小于两个异步任务的 sleep 时间，所以会打印超时的信息。

## 小结

这里我们只是简单入门了一下基于 tokio 的异步任务编程模型，其实 tokio 还提供了很多非常有用的库函数，等待我们在以后继续深挖。
