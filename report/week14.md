## kprobes移植

提取了OS相关部分，包括一些辅助函数与trapframe。文档正在推进。

在移植rcore tutorial的时候发现一个bug：在内核ebreak的时候本来应该直接panic：

```rust
pub fn trap_from_kernel() -> ! {
    panic!("a trap {:?} from kernel!", scause::read().cause());
}
```

但是实际上qemu会直接死机，看寄存器发现PC变成了0。并且这个bug会根据加的println语句随机出现，有时候会正确panic，有时候就会死机。

最后在内核设置stvec的附近断点测试，发现stvec写入后的值是0，导致PC跳到了0。

```rust
pub fn init() {
    set_kernel_trap_entry();
    unsafe{core::arch::asm!("ebreak");}
    println!("trap init done");
}

fn set_kernel_trap_entry() {
    unsafe {
        stvec::write(trap_from_kernel as usize, TrapMode::Direct);
    }
}
```

这是因为stvec是WARL的，写入的时候必须保证低两位合法，而`trap_from_kernel`这个函数的地址没有手动对齐，有概率会导致写入失败。加print语句导致这个函数对齐了的话就会成功。

当然无论如何都是要实现S态到S态的trap的。不过最新版的tutorial有相关代码，直接弄过来就行。

修完以后就可以跑了。