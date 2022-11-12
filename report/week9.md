## 测试程序

本质就是写一个用户程序，然后把它编译好后放在zCore的fs下面运行。我最开始觉得很简单，但是实际上在这块卡了很久。

### 编译

rCore编译是类似于这样的结构

- user
  - lib
    - bpf.c
  - include
    - bpf.h
  - bpftest.c
  - CMakeLists.txt

它实际上是把uCore的头文件都给直接复制到lib和include里面了，然后写一个cmake，这样最后编译的时候就直接cmake就可以了。

这样改的好处在于，如果要加一个系统调用，直接在这个lib里加一个bpf.h 和 bpf.c 就行了。

但是zCore并不是这样的，由于zCore基本上兼容linux，所以它没有 lib 和 include这两个东西，只需要 bpftest.c, 然后它编译的时候直接用的是类似于 `muslgcc bpftest.c -o ...` 这种命令行。那么实际上用的就是musl带带glibc了。

但是这个带来一个bpf实现上的问题。虽然在我们实现的bpf syscall和Linux是完全一样的，在接口上也是完全一样的。并且helper function 也是完全一样的。但是linux的helper function 到 sys_bpf 中间进行了一些处理，这些处理和我们的实现是不一样的，这里可能会导致一些问题。

举个例子，比如 `bpf_create_map` 在linux里就额外经历了一层 `bpf_create_map_xattr` ，而我们的实现中可能就没有这层。那么我们实际上需要在用户态提供头文件之类的把linux原本的helper function覆盖掉。

当然这个比较麻烦，目前我直接ad hoc改了下名字。

### xtask

还有不同的就是zCore里面直接用xtask作为编译的方法。为了加东西，也只能改一下里面的东西。

## eBPF helper functions

### thread/process related

不同于rCore，zCore的调度器并不是一个全局的实例，所以并没有

```rust
use crate::process::current_thread;

current_thread().tid 
```

zCore本身实现时会在syscall层面获取当前的thread引用，因此可以实现getpid等syscall。但是由于eBPF本身是过了syscall这一层，所以就不是很好办。所以对于某些	`helper function: bpf_get_current_pid_tgid` 等就不是很好实现。

目前的办法是从hal层获取当前的thread，然后downcast成zobject thread。

可以看到，rCore与zCore中获取线程的方法与OS本身实现比较耦合，所以我们考虑进行一层封装。即用 ThreadLike这个trait获取所有与线程相关的信息。

目前的办法是：

```rust
use kernel_hal::thread;
use zircon_object::task::Thread;

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

