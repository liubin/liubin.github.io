---
layout: post
title: "Rust async/await 内部是怎么实现的"
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


根据[这里对 `await!`宏](https://rust-lang.github.io/rfcs/2394-async_await.html#generators-and-streams)的说明：

```
let mut future = IntoFuture::into_future($expression);
let mut pin = unsafe { Pin::new_unchecked(&mut future) };
loop {
    match Future::poll(Pin::borrow(&mut pin), &mut ctx) {
          Poll::Ready(item) => break item,
          Poll::Pending     => yield,
    }
}
```

以及[这里](https://rust-lang.github.io/rfcs/2033-experimental-coroutines.html)

```
#[async]
fn print_lines() -> io::Result<()> {
    let addr = "127.0.0.1:8080".parse().unwrap();
    let tcp = await!(TcpStream::connect(&addr))?;
    let io = BufReader::new(tcp);

    #[async]
    for line in io.lines() {
        println!("{}", line);
    }

    Ok(())
}
```

上面代码经过“翻译”后，会类似这样：

```
fn print_lines() -> impl Future<Item = (), Error = io::Error> {
    CoroutineToFuture(|| {
        let addr = "127.0.0.1:8080".parse().unwrap();
        let tcp = {
            let mut future = TcpStream::connect(&addr);
            loop {
                match future.poll() {
                    Ok(Async::Ready(e)) => break Ok(e),
                    Ok(Async::NotReady) => yield,
                    Err(e) => break Err(e),
                }
            }
        }?;

        let io = BufReader::new(tcp);

        let mut stream = io.lines();
        loop {
            let line = {
                match stream.poll()? {
                    Async::Ready(Some(e)) => e,
                    Async::Ready(None) => break,
                    Async::NotReady => {
                        yield;
                        continue
                    }
                }
            };
            println!("{}", line);
        }

        Ok(())
    })
}
```

**Note:** 上面代码 poll 结果还有 NotReady，应该是 RFC 更新不及时吧，最新版的 Future 应该都是 Pendding了。

从上面两处说明，我们也可以大概了解这种 generator 机制了：Ready 的时候返回结果，Pending 的时候让出调度。

今天只是大致搜了下资料，抛出了这样一个问题。下一步计划再确认下 tokio 的实现，看看它到底是怎么做的。
