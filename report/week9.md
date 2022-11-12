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

首先，默认不会编译 test，需要 

`cargo other-test --arch riscv64` 

在编译的时候，它会从环境变量里找到 `riscv64-linux-musl-gcc`，然后枚举所有的 `.c` 

### virtual memory

很奇怪，不知道为啥么不需要过地址转换

## 目前进展

完成了bpf系统调用。可以attach在kprobe上输出对应的context。 这里kprobe根据石圣锋的symbol table进行解析。

```shell
cargo other-test --arch riscv64
cargo qemu --arch riscv64
....
cd bin
./loadprogex.exe
```

期望输出：（用error只是因为颜色比较明显，实际上不是error）

```
file size = 8696
ELF content: 464c457f 10102 0 0
[  5.376442 ERROR 0 0:0 linux_syscall::ebpf] SYS_bpf cmd: 1000, bpf_attr: 1113520, size: 32
[  5.377013 ERROR 0 0:0 linux_syscall::ebpf] load program ex
[  5.377865 ERROR 0 0:0 zircon_object::ebpf::program] bpf program load ex
[  5.378642 ERROR 0 0:0 zircon_object::ebpf::program] map resolution finished
[  5.379326 ERROR 0 0:0 zircon_object::ebpf::program] relocation finished
[  5.381925 ERROR 0 0:0 zircon_object::ebpf::program] compile finished
[  5.382484 ERROR 0 0:0 zircon_object::ebpf] inserted key(fd):1879048192
[  5.383179 ERROR 0 0:0 zircon_object::ebpf::program] OK fd: 1879048192
[  5.383930 ERROR 0 0:0 zircon_object::ebpf::syscall] load ex ret: 1879048192
[  5.384494 ERROR 0 0:0 linux_syscall] syscall result: Ok(1879048192)
load ex: 70000000
target: kprobe$linux_syscall::file::fd::<impl linux_syscall::Syscall>::sys_openat len: 73
bpf prog attach size: 32
[  5.386030 ERROR 0 0:0 linux_syscall::ebpf] SYS_bpf cmd: 8, bpf_attr: 1113520, size: 32
[  5.386557 ERROR 0 0:0 zircon_object::ebpf::syscall] target name str: kprobe$linux_syscall::file::fd::<impl linux_syscall::Syscall>::sys_openat
[  5.387248 ERROR 0 0:0 zircon_object::ebpf::tracepoints] bpf prog attach target: kprobe$linux_syscall::file::fd::<impl linux_syscall::Syscall>::sys_openat 
 fd:1879048192

[  5.388097 ERROR 0 0:0 zircon_object::ebpf::tracepoints] found prog:!
[  5.388548 ERROR 0 0:0 zircon_object::ebpf::tracepoints] prog found!
[  5.389149 ERROR 0 0:0 zircon_object::ebpf::tracepoints] tracepoint parsed: type=KProbe fn=linux_syscall::file::fd::<impl linux_syscall::Syscall>::sys_openat
[  5.389980 ERROR 0 0:0 zircon_object::ebpf::tracepoints] symbol resolved, symbol:linux_syscall::file::fd::<impl linux_syscall::Syscall>::sys_openat addr: ffffffc08023dcbe
[  5.390988 ERROR 0 0:0 zircon_object::ebpf::tracepoints] kprobe registered at addr: ffffffc08023dcbe
[  5.391662 ERROR 0 0:0 zircon_object::ebpf::tracepoints] OK
[  5.392035 ERROR 0 0:0 linux_syscall] syscall result: Ok(0)
```

这部分是 load + attach bpf program

这里是对sys open at 进行hook，可以看到输出之后 warn! 才进行输出，即在函数的第一条指令前运行。

```
triggered!
kprobe  addr = 18446743800981478590
r0  = 15727
r1  = 18446743800981934634
r2  = 18446743801130116720
r3  = 1128448
r4  = 0
r5  = 1113456
r6  = 18446743800981560960
r7  = 315496
r8  = 18446743800986203000
r9  = 18446743800982145720
r10 = 18446743801130116912
r11 = 18446743800985733240
r12 = 18446744073709551516
r13 = 1117672
r14 = 32768
r15 = 0
r16 = 0
r17 = 0
r18 = 18446743800985733320
r19 = 18446743800985733264
r20 = 18446743800985733240
r21 = 18446743800986203000
r22 = 18446743800985733120
r23 = 18446743801130122976
r24 = 18446743800984095808
r25 = 0
r26 = 18446743801130122976
r27 = 18446743800985733264
r28 = 111148
r29 = 32
r30 = 121
r31 = 114
[  5.396368 WARN  0 0:0 linux_syscall::file::fd] sys openat called!
```



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

这个在linux里面实际上是像一个pseudo file写东西，类似于管道，作为内核log的输出方式。但是rCore直接是用了print! 这个宏来做。在zCore中使用 kernel_hal::console 进行输出。

## refactor bpf map

`syscall.rs` 是与OS相关的部分，这里采用linux的接口。

其他部分是eBPF内部，使用内部定义的 `retcode.rs`, `const.rs` 中的enum，使用 `BpfResult` 作为返回值， 避免常量污染。

但是目前还没有完全完成。

## todo

因为遇到了意想不到的问题，所以有些地方比较赶，所以优化下。

bpf map 功能

添加更多 helper functions，主要是bpf map的。

