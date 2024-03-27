# [Async Rust] How does Tokio's IO work?

I've been reading a book lately. Quite a good book: [Asynchronous Programming in Rust](https://www.packtpub.com/product/asynchronous-programming-in-rust/9781805128137).
It explains how asynchronous programming works using [Rust](https://www.rust-lang.org/) as an example. Writing in Rust is my long-term hobby, so I'm trying to get as deep into it as possible. From some other books, research and podcasts I know, that it's active recall and creating your own notes make you remember things. 
I'll be posting my notes in there, as I discovered a lot of fun stuff and definitions.


## Concurrency vs Parallelism

Concurrency is about making things efficient, parallelism is throwing more resources at tasks.

Tasks can be parallel, but not concurrent.
Tasks can be concurrent, but not parallel.

## Green threads vs coroutines

Green threads have stack and can be preempted at any point in time.
Coroutines are state machines and can only be preempted at fixed points (await points).

Golang works on green threads, Rust async runtimes on coroutines.
Coroutines are likely to be cooperative, green threads not so much.

## Why do we need async I/O?

When we want to read data from a resource, e.g. bytes from a network connection we need to make a syscall. If we use [`std::net::TcpStream`](https://doc.rust-lang.org/std/net/struct.TcpStream.html) directly, when we perform a `stream.read()` we make a syscall, park the thread and wait for it to be waken by OS when the data is ready. If we got multiple connections, we need to have multiple threads and they're expensive.

There is non-blocking (threads are not parked) way of getting data. OSes have different ways of doing that, Linux has [`epoll`](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html), Windows `iocp`, MacOS `kqueue`.
Basically we create a queue in the OS and subscribe to it.
One thread creates multiple connections, event queue, subscribes to it, and only blocks once (after all of the subscribtions have been made).
This way, while we wait for the data to come, we can continue to make other work.

There is a nice library, which tokio is based upon, called [`mio`](https://github.com/tokio-rs/mio). In the book, I've been implementing a simplified version of mio.

Link to the full example: [github.com/th7nder/my-mio](https://github.com/th7nder/my-mio/blob/main/src/main.rs).
[Delay Server](https://github.com/PacktPublishing/Asynchronous-Programming-in-Rust/tree/main/delayserver)
Example excerpt:
```rust

 for stream_id in 0..NUMBER_OF_STREAMS {
    let mut stream = TcpStream::connect(DELAYSERVER_ADDR)?;
    stream.set_nonblocking(true)?;

    stream.write_all(
        delayserver_request(
            (NUMBER_OF_STREAMS - stream_id) * 1000,
            format!("request-{stream_id}").as_str(),
        )
        .as_bytes(),
    )?;

    poll.registry()
        .register(&stream, stream_id, ffi::EPOLLIN | ffi::EPOLLET)?;
    streams.push(stream);
}

eprintln!("Created connections... processing events");
let expected_events = NUMBER_OF_STREAMS;
let mut handled_events = 0;
while handled_events < expected_events {
    let mut events = Vec::with_capacity(10);
    poll.poll(&mut events, None)?;
    eprintln!("Processing: {} events", events.len());
    if events.is_empty() {
        println!("TIMEOUT (OR SPURIOUS EVENT NOTIFICATION)");
        continue;
    }
    
    handled_events += handle_events(&events, &mut streams)?;
}
```