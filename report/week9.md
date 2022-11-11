## eBPF helper functions

### thread/process related

不同于rCore，zCore的调度器并不是一个全局的实例，所以并没有

```rust
use crate::process::current_thread;

current_thread().tid 
```

这种直接获取当前线程的接口。而是需要从hal层里 downcast出来。

当然，这种直接获取的做法也不是很优雅，所以我们考虑进行一层封装。

所以对于某些	`helper function: bpf_get_current_pid_tgid` 等就不是很好实现。

zCore本身实现时会在syscall层面获取当前的thread引用，因此可以实现getpid等syscall。但是由于eBPF本身是过了syscall这一层，所以就不是很好办。

目前的办法是：

```rust
use kernel_hal::thread;

pub trait ThreadLike : Sync + Send {
    fn get_pid(&self) -> u64;
    fn get_tid(&self) -> u64;
    ...
}


impl ThreadLike for Thread {
    fn get_pid(&self) -> u64 {
        return self.proc().id();
    }
    fn get_tid(&self) -> u64 {
        return self.related_koid() as u64;
    }
    ...
}

pub fn os_current_thread() -> Arc<dyn ThreadLike> {
    if let Some(thread) = kernel_hal::thread::get_current_thread() {
        let ret = thread.downcast::<Thread>().unwrap();
        ret
    } else {
        panic!("cannot get current thread!")
    }
}
```

这样的话，helper function就可以直接调用 os_current_thread，这部分代码就可以单独封装了。

### bpf_trace_printk

这个在linux里面实际上是像一个pseudo file写东西，类似于管道，作为内核log的输出方式。但是rCore直接是用了print! 这个宏来做。在zCore中也可以通过类似的方法做，不知道合不合适。



## refactor bpf map

`syscall.rs` 是与OS相关的部分，这里采用linux的接口。

其他部分是eBPF内部，使用内部定义的 `retcode.rs`, `const.rs` 中的enum，使用 `BpfResult` 作为返回值， 避免常量污染。