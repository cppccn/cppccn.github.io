---
layout: default
title:  "📉 Asynchronous futures vs Synchronous locks"
description: "When we're working on a project, and we want to accelerate the process, we may think about parallelism and concurrency...
<a href='./2022/01/14/benchmark-async.html'>continue to read</a>"

authors: ["Adrien Zinger"]
reviewers: ["Yvan Sraka"]
---

# 📉 Asynchronous futures vs Synchronous locks

When we're working on a project, and we want to accelerate the process, we may think about parallelism and concurrency. A few days ago, I wrote this unit test:

```rust
#[tokio::test]
async fn tokio_test_bench() {
    // Test if dyn-timeout spend at least 40 milliseconds before running
    // the callback function
    {
        let mut time = TIME.lock().unwrap();
        *time = SystemTime::now();
    }
    let mut dyn_timeout = tokio_impl::DynTimeout::new(TWENTY, move || {
        // *Synchronous* callack
        let st = TIME.lock().unwrap();
        let dur = st.elapsed().unwrap();
        assert!(dur > Duration::from_millis(36) && dur < Duration::from_millis(44));
    });
    dyn_timeout.add(TWENTY).await.unwrap();
    dyn_timeout.wait().await.unwrap();
}
```

We can discuss eventually about the quality of this code, I'm pretty sure that we can think it's weird. Well, I think it's pretty unusual because to test this, I need to use a synchronous variable, which I lock synchronously and then continue the job. It's just a unit test, so I can justify the odd design by the correctness.

But yesterday, I looked at a comment in the _Rust for Rustacean_ book. The comment said that, I should be careful because of the waste of time when I wait synchronous code in a future. _"That's exactly my problem!!" - me in the train_


## Reproduce the problem and check

The reason why we don't want to wait a synchronous code in a future, it's because it may block for a long time the current thread and erase the benefice of the futures management. To differentiate concurrency and parallelism, Jon Gjengset wrote this ASCII art: concurrency `_-_-_-`, parallelism `====`. This graphical example can help us to understand the next thing: synchronous lock in concurrency.

In my unit test, I have one single thread who works. And I want to be sure that my timeout is waiting 40 milliseconds, but to know this value, I need to pass a value (here the TIME static value) from the main future to the synchronous callback [^1]. So I cannot have a tokio `Mutex` here and just do the same thing with a simple `.await` after my lock. Let's say that I want suddenly spawn N timeouts, how my single thread will react?

```rust
#[tokio::main]
async fn main() {
    let mut f = vec![];
    for _ in 0..N {
        // From 0 to N, create a new future that launch a timeout
        f.push(async move {
            {
                let mut time = TIME.lock().unwrap();
                time.push_back(SystemTime::now());
            }
            let mut dyn_timeout = tokio_impl::DynTimeout::new(TWENTY, || {
                let st = TIME.lock().unwrap().pop_front().unwrap();
                let dur = st.elapsed().unwrap();
                println!("{}", dur.as_millis());
            });
            dyn_timeout.wait().await.unwrap();
        });
    }
    // Join all the spawned futures before the drop
    join_all(f.into_iter()).await;
}
```

If I suddenly want to spawn N timeouts, I guess it's correct to think that my code will wait too much in the `Mutex`. I will block the other concurrent green threads and make the code really slow. Yes, because now, the concurrency in ASCII art will be hard to wrote with two characters, but I'll try:

```
- `         -------------             `
- `---------                          `
- `                      -------------`
- etc N times
```

We can deduce one thing right now, even if we had full access to the timer, as fast as we can, the concurrency design already get us in a latency. And increasing the number of timeouts concurrent in the same thread increase randomly (but certainly) the latency.

Back to my current thread who manage my futures, each times one of these is awaiting, the runtime processes to run another future ready to run. And now let's draw how the synchronous lock change it.


```
- `            -----xxxxxx--------                   `
- `---xxx------                                      `
- `                               ---xxxxxx----------`
- etc N times
```

Definitively, the latency will increase again, and the concurrency will not help us. The `xxxx` isn't a signal to our runtime that the current futures can be polled again, the runtime continue to process the execution while it has the access to the static arc and release after the job done. With this code, we obtained a medium value of 45.1209 instead of 20 milliseconds with 10k timeouts.

This is why we want a new design, in order to avoid this heavy CPU lock and continue with a fast concurrency design.


## Re-design the test, full futures

In this design, I just want to send timeout and receive handle it with the help of futures. I want to forget the badly designed synchronous lock in my algorithm!

```rust
#[tokio::main]
async fn main() {
    let (sender, receiver) = &mut tokio::sync::mpsc::channel::<()>(1);
    // The sender and receiver here is used by `new_notif` to notify that
    // a timeout has been thrown between the asynchronous executions.
    
    let sender_s = sender.clone();
    let times: Arc::<tokio::sync::Mutex::<VecDeque<SystemTime>>> = Default::default();
    let times_t = times.clone();
    
    // Spawn the timeouts in the runtime and continue
    tokio::spawn(async move {
        // Later, start to spawn timeouts
        for _ in 0..N {
            let sender_notif = sender_s.clone();
            let curr_times = times_t.clone();
            tokio::spawn(async move {
                {
                    let mut time = curr_times.lock().await;
                    time.push_back(SystemTime::now());
                }
                let mut dyn_timeout = tokio_impl::DynTimeout::new_notif(TWENTY, sender_notif, || {});
                dyn_timeout.wait().await.unwrap();
            });
        }
    });
    
    // Start handling  
    for _ in 0..N {
        match receiver.recv().await {
            Some(_) => {
                let st = times.lock().await.pop_front().unwrap();
                let dur = st.elapsed().unwrap();
                println!("{}", dur.as_millis());
            }
            _ => break,
        }
    }
}
```

I hacked a bit of the `dyn-timeout` project to add an `async` message passing. After all, the function given in argument is executed in a future, so I just have to make this timeout executing a new future that send the message! Now I'm pretty sure that, instead of `xxxx`, my current thread will yield and will be ready when `times` is usable. No more synchronous heavy lock! I run it and.... wait, what? I obtain a medium value of 153 milliseconds latency for 10k timeouts.

As I wrote in the first chapter, the concurrency by design is the source of latency when we absolutely want precision in a timeout. Passing a message here seems to be heavy and a lot slower than his dirty synchronous cousin. Nevertheless, I'm thinking about another explanation. Imagine that in your runtime [^2] you spawn suddenly 10k timeout, it means that you want to run at least 20k asynchronous code. Yes, one part of the asynchronous is before the timeout, and the other is after. Then in our example, we add a part to handle the timeout, to signal the timeout and finally to access to the `times` variable (two times). Of course, it's one of the reason of the latency.


## Split the work

Another architecture, more convenient, is to separate what can be separated. Clearly, the handling of the values can be in another thread. It'll theoretically accelerate the execution. Theoretically, it will take a half of the asynchronous code in the thread timeout and reduce the latency.

```rust
#[tokio::main]
async fn main() {
    //faster
    let (sender, receiver) = &mut tokio::sync::mpsc::channel::<()>(100000);
    let sender_t = sender.clone();
    let times: Arc::<tokio::sync::Mutex::<VecDeque<SystemTime>>> = Default::default();
    let times_t = times.clone();

    let th = std::thread::spawn( move || {
        let rt1 = tokio::runtime::Runtime::new().unwrap();
        let sender_s = sender_t.clone();
        rt1.block_on(async move {
            let mut futures = vec![];
            for _ in 0..N {
                let sender_notif = sender_s.clone();
                let curr_times = times_t.clone();
                futures.push(async move {
                    {
                        let mut time = curr_times.lock().await;
                        time.push_back(SystemTime::now());
                    }
                    let mut dyn_timeout = tokio_impl::DynTimeout::new_notif(TWENTY, sender_notif, || {});
                    dyn_timeout.wait().await.unwrap();
                });
            }
            join_all(futures).await;
        });
    });

    for _ in 0..N {
        match receiver.recv().await {
            Some(_) => {
                let st = times.lock().await.pop_front().unwrap();
                let dur = st.elapsed().unwrap();
                println!("{}", dur.as_millis());
            }
            _ => break,
        }
    }
    th.join().unwrap();
}
```

It's going to take a lot of space in the CPU, but for the best! Now we have two workers, one with the timeout, another for handling it. After some test, I get a medium value of 84 milliseconds! Oh... 84? The code is hard to read, take two threads, has the biggest channel of all time, and is slower than a heavy synchronous lock in the thread.


## Summary

I looked at different architecture that can be debated. Spawning 10k futures seems to not be a common problem. Nevertheless, in a big project, we don't always have hands on all the code each time we're programming. Sometimes, a latency of 1000 percent can be invisible. And when you start spotting the issue, you don't know where to look at first. In the life of your project, you can reach more than 10k futures, and you have to know what it means for your code.

The obvious thing to do in a project, is to test the execution speed when you have a critical amount of task to execute. When you want to proceed to a change of the concurrency/parallelism architecture because of something that you read said _"You should do that instead of that"_, be careful and implement a benchmark first.

![Graph of latencies](/assets/Spawn 10k timeouts of 20 milliseconds.svg)

You can see the single thread with full concurrency as the slower algorithm. The dual threaded concurrency algorithm instead has a lower latency with a slow amount of timeouts and increase suddenly. The initial algorithm continues to be the one with the less latency.



---

[^1]: The reason why this callback is synchronous is because I couldn't safelly send a future between spawned thread and the design may be rethinked in a while.
[^2]: By runtime I mean the tokio runtime that execute concurrency
