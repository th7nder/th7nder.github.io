### [Async Rust] Anatomy of fibers 

TL;DR:
Whole example available at: [th7nder/my-fibers](https://github.com/th7nder/my-fibers).

This is just another post from my learning about the building blocks of concurrency, based on the book [Asynchronous Programming in Rust](https://www.packtpub.com/product/asynchronous-programming-in-rust/9781805128137). 

There are ways in which your program can be concurrent, be it fibers (stackful coroutines) or futures (stackless couroutines).

It turns out that fibers, are just a fancy way of implementing the same thing the kernel does (threads), but on our own. With usage of some inline assembly, registers, ABIs, allocating stacks, yielding and all of the machinery.
It can be more efficient, because we don't give control to the OS when we yield, we just give control to our own threads and OS does not even know about them.
We can maximize the CPU time in our program, by not giving control back to the OS. 

The most fun port of this challenge was understanding assembly and implementing my own context switch:

```rust
#[naked]
#[no_mangle]
#[cfg_attr(target_os = "macos", export_name = "\x01switch")]
unsafe extern "C" fn switch() {
    asm!(
        "mov [rdi + 0x00], rsp",
        "mov [rdi + 0x08], r15",
        "mov [rdi + 0x10], r14",
        "mov [rdi + 0x18], r13",
        "mov [rdi + 0x20], r12",
        "mov [rdi + 0x28], rbx",
        "mov [rdi + 0x30], rbp",
        "mov rsp, [rsi + 0x00]",
        "mov r15, [rsi + 0x08]",
        "mov r14, [rsi + 0x10]",
        "mov r13, [rsi + 0x18]",
        "mov r12, [rsi + 0x20]",
        "mov rbx, [rsi + 0x28]",
        "mov rbp, [rsi + 0x30]",
        "ret", options(noreturn)
    );
}

```

As well as my first-ever round-robin scheduler:

```rust
    #[inline(never)]
    fn t_yield(&mut self) -> bool {
        let mut pos = self.current;

        while self.threads[pos].state != State::Ready {
            pos += 1;
            if pos == self.threads.len() {
                pos = 0;
            }

            if pos == self.current {
                // no threads are ready, quitting the runtime
                return false;
            }
        }

        if self.threads[self.current].state != State::Available {
            self.threads[self.current].state = State::Ready;
        }

        self.threads[pos].state = State::Running;
        let old = self.current;
        self.current = pos;

        let old_context: *mut ThreadContext = &mut self.threads[old].context;
        let new_context: *const ThreadContext = &mut self.threads[self.current].context;
        unsafe {
            asm!("call switch", in("rdi") old_context, in("rsi") new_context, clobber_abi("C"));
        }

        self.threads.len() > 0
    }
```