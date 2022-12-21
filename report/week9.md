## kprobe移植
给zCore添加了Symbol table，一开始是把symbol table作为文本文件放在了根目录，然后尝试init的时候读这个文件，但是因为zCore架构的原因，内核态去读一个特定文件有点麻烦，所以目前还是用rCore的方法，在编译以后直接把symbol table写到elf的特定区域里，init的时候去读。

实现了符号表以后，kprobe，kretprobe，trace三个模块就都可以正常运行了，可以算移植完毕。

初始化时会进行一些测试，包含之前同学写的所有测试以及通过符号表挂载的测试，效果如下：
```
[  0.096921 WARN  0 0:0 zircon_object::symbol::table] Loading kernel symbol table ffffffc08031c000 with size 00028096
[  0.125410 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_PRE_HANDLER] entering fn kprobes_test1, addr = 0xffffffc0802001a2, pc = 0xffffffc0802001a2
[  0.126256 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_POST_HANDLER] post handler invoked
[  0.126927 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_PRE_HANDLER] entering fn kprobes_test2, addr = 0xffffffc0802001b6, pc = 0xffffffc0802001b6
[  0.127429 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_POST_HANDLER] post handler invoked
[  0.128041 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_PRE_HANDLER] entering fn kprobes_test3, addr = 0xffffffc0802001ca, pc = 0xffffffc0802001ca
[  0.128529 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_POST_HANDLER] post handler invoked
[  0.129008 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_PRE_HANDLER] entering fn kprobes_test4, addr = 0xffffffc0802001e8, pc = 0xffffffc0802001e8
[  0.129575 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_POST_HANDLER] post handler invoked
[  0.129984 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_PRE_HANDLER] entering fn kprobes_test5_entry, addr = 0xffffffc080200208, pc = 0xffffffc080200208
[  0.130485 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_POST_HANDLER] post handler invoked
[  0.131570 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] entering fn, a0 = 1
[  0.135292 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] in recursive_fn(1)
[  0.135644 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] entering fn, a0 = 2
[  0.136337 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] in recursive_fn(2)
[  0.136672 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] entering fn, a0 = 3
[  0.137242 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] in recursive_fn(3)
[  0.137607 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] entering fn, a0 = 4
[  0.137978 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] in recursive_fn(4)
[  0.138343 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] entering fn, a0 = 5
[  0.138974 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] exiting fn, a0 = 100
[  0.139742 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] exiting fn, a0 = 104
[  0.140144 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] exiting fn, a0 = 107
[  0.140578 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] exiting fn, a0 = 109
[  0.140972 WARN  0 0:0 zircon_object::probe::tests::kretprobes_test] exiting fn, a0 = 110
[  0.143634 WARN  0 0:0 zircon_object::probe::tests::trace_test] function zircon_object::probe::tests::trace_test::foo at depth 0)
[  0.144145 WARN  0 0:0 zircon_object::probe::tests::trace_test] function zircon_object::probe::tests::trace_test::Test::l0 at depth 0)
[  0.144643 WARN  0 0:0 zircon_object::probe::tests::trace_test] function zircon_object::probe::tests::trace_test::Test::l1_1 at depth 1)
[  0.145119 WARN  0 0:0 zircon_object::probe::tests::trace_test] function zircon_object::probe::tests::trace_test::Test::l1_2 at depth 1)
[  0.145618 WARN  0 0:0 zircon_object::probe::tests::trace_test] function zircon_object::probe::tests::trace_test::Test::l2_3 at depth 2)
[  0.146064 WARN  0 0:0 zircon_object::probe::tests::trace_test] function zircon_object::probe::tests::trace_test::Test::l2_4 at depth 2)
[  0.146581 WARN  0 0:0 zircon_object::probe::tests::trace_test] function zircon_object::probe::tests::trace_test::Test::l2_1 at depth 2)
[  0.147072 WARN  0 0:0 zircon_object::probe::tests::trace_test] function zircon_object::probe::tests::trace_test::Test::l2_2 at depth 2)
```

同时会在mkdir的syscall上挂一个probe，打印出来的结果如下：

```
/ # mkdir test
[  6.272002 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_PRE_HANDLER] entering fn linux_syscall::file::dir::<impl linux_syscall::Syscall>::sys_mkdirat, addr = 0xffffffc08023c9d2, pc = 0xffffffc08023c9d2
[  6.272531 WARN  0 0:0 zircon_object::probe::tests::kprobes_test] [KPROBE_POST_HANDLER] post handler invoked
```

backtrace理论上也可以运行，但是这个依赖于frame_pointer，之前rCore里是用`"eliminate-frame-pointer": false`这个flag设置的，但是新版本编译器已经不认这个了，目前还没有解决。
