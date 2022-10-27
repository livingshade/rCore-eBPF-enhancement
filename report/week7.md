## eBPF 模块化

初步想法，eBPF可以分为三个部分

- kprobe： 用于提供context
- memory： 用于存储bpf program
- fs： 用于存储bpf map，以及持久化bpf program

#### kprobe

os相关，希望能提供类似以下的接口

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


fn load_bpf_program_to_kprobe(hook_point: HookPoint, bpf_program: BPFProgram) -> kProbe {
  let kprobe = kProbe::from(hook_point);
  kprobe.set_handler(bpf_program.get_bytecode());
}

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
  fn init_self(t: <KernelAllocator>, code: ..) {
    self.verify(code); // JIT compiler
    self.addr = t.kalloc(self.byte.size());
    memcpy(..)
  }
}

fn sys_bpf() -> BPFProgram {
  ...
}
```

### fs

提供bpf map的相关内容，是实现前两者模块化后的内容。

## zCore

有几个问题需要杨德睿助教解答下，群里问了。

- kernel allocator需要实现
- 需要把rvjit等一套加进去，作为crate？

