### [Async Rust] Async Runtimes in 5 minutes

I've finished my recently most favourite book: [Asynchronous Programming in Rust](https://www.packtpub.com/product/asynchronous-programming-in-rust/9781805128137). 
It's been a blast and I understood what async runtimes are and how they work.
I wanted to summarize it, so I'll have a reference to it and solifidy my newly acquired knowledge . 

## So, how do runtimes work?

Rust standard library provides a `Future` trait. A future, represents a value that will be ready in future. When we create a future, we create a struct that'll represent the progress and the state of the value the future should provide. 

This trait is nothing special in itself:

```rust
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

We can implement it easily by ourselves (skipping reactor logic):

```rust
impl Future for HttpGetFuture {
    type Output = String;

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        let id = self.id;
        let this = self.get_mut();
        if this.stream.is_none() {
            this.write_request();
            let stream = this.stream.as_mut().unwrap();
        }

        let mut buf = [0u8; 4096];
        loop {
            match this.stream.as_mut().unwrap().read(&mut buf) {
                Ok(0) => {
                    let str = String::from_utf8_lossy(&this.buffer);
                    break Poll::Ready(str.to_string());
                }
                Ok(n) => {
                    this.buffer.extend(&buf[..n]);
                }
                Err(k) if k.kind() == ErrorKind::WouldBlock => {
                    break Poll::Pending;
                }
            }
        }
    }
}
```

Our future is representing a future HTTP response.
When something calls poll, it's initialized and performs a TCP connection. When there is no response, it returns with Poll::Pending.
Then something needs to call it again, when some data is available. 
There are two somethings in this story. **Executor** and **Reactor** connected by a **Waker**.

**Executor** is a runtime scheduler which calls `.poll()` on the future to make it progress and it goes to sleep when a future returns `Poll::Pending`:

```rust
loop {
    while let Some(ready_id) = self.pop_ready() {
        let mut future = match self.get_future(ready_id) {
            Some(f) => f,
            None => continue,
        };

        let waker: Waker = self.get_waker(ready_id).into();
        let mut cx = Context::from_waker(&waker);
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(_) => continue,
            Poll::Pending => {
                self.insert_task(ready_id, future);
                continue;
            }
        }
    }
    if self.task_count() > 0 {
        println!("Waiting for {task_count} tasks");
        thread::park();
    } else {
        println!("Finished, exiting...");
        break;
    }
}


```

**Reactor** is listening to a system event queue (epoll, kqueue, IOCP) and wakes the **Executor** thread when an event is ready. **Executor** wakes and polls the future again. It has an event loop, which is blocking on `poll.poll` and when something happens it calls an appropriate waker. 


```rust
fn event_loop(mut poll: Poll, wakers: Wakers) {
    let mut events = Events::with_capacity(32);
    loop {
        poll.poll(&mut events, None).unwrap();
        for event in events.iter() {
            let Token(id) = event.token();
            let wakers = wakers.lock().unwrap();
            if let Some(waker) = wakers.get(&id) {
                waker.wake_by_ref();
            }
        }
    }
}
```

**Waker** in itself is just an object which remembers which thread to `unpark`. 

```rust
impl Wake for MyWaker {
    fn wake(self: Arc<Self>) {
        self.ready_queue
            .lock()
            .map(|mut q| q.push(self.id))
            .unwrap();

        self.thread.unpark();
    }
}
```

That's it, full example at: [github.com/th7nder/my-runtime](https://github.com/th7nder/my-runtime).

Btw. it took years of implementation and discussions to come-up with this design, what I tried to summarize is just a bird's eye overview, but it essentially boils down to it. If you wanna know more, reach out to me or read the [book](https://www.packtpub.com/product/asynchronous-programming-in-rust/9781805128137)! It's amazing.