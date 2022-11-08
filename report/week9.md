## eBPF helper functions

### thread/process related

不同于rCore，zCore的调度器并不是一个全局的实例，所以并没有

```rust
use crate::process::current_thread;

current_thread().tid 
```

这种直接获取当前线程的接口。

所以对于某些	`helper function: bpf_get_current_pid_tgid` 等就不是很好实现。

zCore本身实现时会在syscall层面获取当前的thread引用，因此可以实现getpid等syscall。但是由于eBPF本身是过了syscall这一层，所以就不是很好办。

目前的办法是：

### bpf_trace_printk

这个在linux里面实际上是像一个pseudo file写东西，类似于管道，作为内核log的输出方式。但是rCore直接是用了print! 这个宏来做。在zCore中也可以通过类似的方法做，不知道合不合适。

