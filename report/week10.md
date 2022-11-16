## async probing
### backtrace
给rustc添加`force-frame-pointers=yes`开启栈帧。

关于backtrace的符号问题，[这里](https://github.com/rust-lang/rust/issues/65978)的一个回复解释了：
`This is caused by the legacy (but still default) symbol mangling rustc uses. If you compile with -Csymbol-mangling-version=v0, you should get much more detailed symbol names.`。

所以加上这个选项就可以了，可以发现`<<core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll>`变成了
`<core::future::from_generator::GenFuture<zcore_loader::linux::run_user::{closure#0}> as core::future::future::Future>::poll`。

```
panicked at 'backtrace test', zircon-object/src/probe/mod.rs:94:5
=== BEGIN zCore stack trace ===
#00 PC: 0xFFFFFFC08028DF96 FP: 0xFFFFFFC0BF07E5C0
    rust_begin_unwind +0x2a6
#01 PC: 0xFFFFFFC080206B54 FP: 0xFFFFFFC0BF07E600
    core::panicking::panic +0x40
#02 PC: 0xFFFFFFC0802DA4C8 FP: 0xFFFFFFC0BF07E650
    zircon_object::probe::backtrace_test +0x22
#03 PC: 0xFFFFFFC0802C51AA FP: 0xFFFFFFC0BF07E660
    <linux_syscall::Syscall>::syscall::{closure#0} +0x8bf8
#04 PC: 0xFFFFFFC0802BBD10 FP: 0xFFFFFFC0BF07FAC0
    <core::future::from_generator::GenFuture<zcore_loader::linux::run_user::{closure#0}> as core::future::future::Future>::poll +0xe64
#05 PC: 0xFFFFFFC0802D7A8E FP: 0xFFFFFFC0BF07FDB0
    <zircon_object::task::thread::ThreadSwitchFuture as core::future::future::Future>::poll +0x1e0
#06 PC: 0xFFFFFFC08020C788 FP: 0xFFFFFFC0BF07FE10
    run_executor +0x4a8
#07 PC: 0xFFFFFFC080200008 FP: 0xFFFFFFC0BF080000
    executor_entry
=== END zCore stack trace ===
```

但是因为一些函数被inline，syscall可能会变成`<linux_syscall::Syscall>::syscall::{closure#0} +0x8bf8`这种不明确的符号。

### probe
与backtrace遇到的问题一样，inline的函数没法trace，但是可以通过llvm的debuginfo来找inline优化到的的具体位置，这些信息都在生成的DWARF格式的文件里，里面也有async函数生成的一些中间量。

就是这种东西：
```
DW_TAG_inlined_subroutine
    DW_AT_abstract_origin	(0x00582841 "_RNvXs_NvNtCs96tDTpbUReL_4core6future14from_generatorINtB4_9GenFutureNCNvMNtNtCs89qvxiTQOvg_13linux_syscall4file4fileNtB1d_7Syscall8sys_read0ENtNtB6_6future6Future4pollCsagZkAU4YrCP_12zcore_loader")
    DW_AT_ranges	(0x00124660
        [0xffffffc0802bce94, 0xffffffc0802bce98)
        [0xffffffc0802bce9e, 0xffffffc0802bced4)
        [0xffffffc0802bda42, 0xffffffc0802bdab2)
        [0xffffffc0802bed68, 0xffffffc0802bedc8)
        [0xffffffc0802c4f16, 0xffffffc0802c4f3a)
        [0xffffffc0802c4f7a, 0xffffffc0802c4f8c))
    DW_AT_call_file	("/home/s2020012692/ebpf/zCore/linux-syscall/src/lib.rs")
    DW_AT_call_line	(82)
    DW_AT_call_column	(0x41)
```

既然gdb在optimized的情况下可以用这些debuginfo对应到行号，那probe肯定是可以的，相当于内核里内嵌自己的debuginfo以及gdb的部分功能。

从这里找async的中间过程应该会清晰一些，因为被优化掉的符号都出来了，还在研究。

## zCore
帮刘圣debug zCore的地址问题，同时教刘圣怎么使用键盘。
