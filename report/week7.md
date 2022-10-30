## eBPF 模块化

初步想法，eBPF可以分为三个部分

- kprobe： 用于提供context
- memory： 用于存储bpf program
- fs/bpf： 用于存储bpf map，以及持久化bpf program

### Context

os相关，希望能提供类似以下的接口, 对于kprobe而言就是寄存器

```rust
use eBPF::BPFProgram;
struct HookPoint;
struct kProbe;

impl for kProbe {
  fn set_handler(self: &Self, addr: usize, len: usize) -> () {
    	self.entry = addr;
    	// do some thing on addr + len to return to normal control flow
    ...
  }
}

```

实际上不应该仅仅是kprobe，应当是对event（linux在网络栈上还支持很多）

https://blogs.oracle.com/linux/post/bpf-a-tour-of-program-types

```C
enum bpf_prog_type {
    BPF_PROG_TYPE_UNSPEC,
    BPF_PROG_TYPE_SOCKET_FILTER,
    BPF_PROG_TYPE_KPROBE,
    BPF_PROG_TYPE_SCHED_CLS,
    BPF_PROG_TYPE_SCHED_ACT,
    BPF_PROG_TYPE_TRACEPOINT,
    BPF_PROG_TYPE_XDP,
    BPF_PROG_TYPE_PERF_EVENT,
    BPF_PROG_TYPE_CGROUP_SKB,
    BPF_PROG_TYPE_CGROUP_SOCK,
    BPF_PROG_TYPE_LWT_IN,
    BPF_PROG_TYPE_LWT_OUT,
    BPF_PROG_TYPE_LWT_XMIT,
    BPF_PROG_TYPE_SOCK_OPS,
    BPF_PROG_TYPE_SK_SKB,
};
```



### bpf(2) 接口

```rust
fn sys_bpf() -> i32 {
  
}
// helpers
fn bpf_prog_load() -> fd {
  ...
}

fn load_bpf_program_to_kprobe(hook_point: HookPoint, bpf_program: BPFProgram) {
  let kprobe = kProbe::from(hook_point);
  kprobe.set_handler(bpf_program.get_bytecode());
}


/* 
int   bpf_prog_load(enum bpf_prog_type type,
                         const struct bpf_insn *insns, int insn_cnt,
                         const char *license)
           {
               union bpf_attr attr = {
                   .prog_type = type,
                   .insns     = ptr_to_u64(insns),
                   .insn_cnt  = insn_cnt,
                   .license   = ptr_to_u64(license),
                   .log_buf   = ptr_to_u64(bpf_log_buf),
                   .log_size  = LOG_BUF_SIZE,
                   .log_level = 1,
               };

               return bpf(BPF_PROG_LOAD, &attr, sizeof(attr));
           }
 */

fn bpf_map_create() .. 
```

### memory

提供bpf program在内存中的位置，类似于

```rust
trait KernelAllocator {
  fn kalloc(len: usize) -> Option(usize) {
    
  }
  fn kfree(addr: usize) {
    
  }
}

// bpf need os to provide an allocator 

impl for BPFProgram {
  fn init_self(t: <KernelAllocator>, bpf_attr: BPFattr) {
    self.verify(bpf_attr); // JIT compiler
    self.addr = t.kalloc(self.byte.size());
    memcpy(..)
  }
}

struct fd(i32);

```

### 虚拟文件系统

对于linux，一个bpf map之后，实际上会在 `/sys/fs/bpf/..` 看到对应的map，虽然这个并不是实际意义上的文件。目前的实现只是有一个fd，还需要对接到fs，不过这个应该是后面的内容。



## zCore

- kernel allocator需要实现

- eBPF模块需要jit的部分，这里应该可以原样使用。

- 需要看懂代码。

  ```rust
  pub fn frame_alloc() -> Option<PhysAddr> 
  ```

  

