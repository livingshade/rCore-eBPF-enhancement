## kprobe模块研究
kprobe实现为将需要probe的位置的指令改成ebreak，走过去的时候触发trap，从而handler能够知道这里要probe。

probe函数能获取的信息就是这次ebreak的trapframe以及创建probe时用户设定的数据。probe结束后需要模拟原来指令的执行，这个指令如果需要模拟，则结果反映在trapframe里，否则可以直接在一片trampoline中执行。执行后可以有一个post_handler执行，这里需要根据是否模拟指令作不同的处理。

现在对于模拟的指令也开了buffer，实际上应该是不需要的，只是实现比较方便。
## 解决问题
rCore可以编译，装了一下`rustfilt`。

修复rust-analyzer的问题，在`kernel`里开`.cargo/config.toml`写上`target = "riscv64gc-unknown-linux-musl"`就好。

目前rcore跑ucore的程序会出现unhandled page fault(SIGSEGV)，对应rCore代码里是一个TODO，但是按理说不应该段错误，正在解决。

不能对同一个地址注册多个kprobe，所以多核时test会有一些注册失败的error，不过多核执行一个probe没有问题。
`As of Linux v2.6.15-rc1, multiple handlers (or multiple instances of the same handler) may run concurrently on different CPUs. Kprobes does not use mutexes or allocate memory except during registration and unregistration.`

kprobe与handler返回值的交互
`If you change the instruction pointer (and set up other related registers) in pre_handler, you must return !0 so that kprobes stops single stepping and just returns to the given address. This also means post_handler should not be called anymore.`
暂时没有实现，只考虑执行eBPF函数的话是否需要这个？

rust库可以把unsafe块包装成safe的函数，从而依赖该库的程序调用的时候不会察觉到😅

## zCore阅读
看了一下毕设论文，里面的代码基本都被迭代完了，只能大概看一下架构。

zCore的系统调用都是async的，如果要动态跟踪需要实现对async的probe，静态追踪可以考虑用rust宏实现在每个syscall里插入probe？

## rust async
研究了一下rust的async，先看的[官方book](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html).。

比较难理解的是`futures do nothing unless you .await or poll them`这句话。Poll返回一个状态表示是否完成，那么我poll一次他应该执行多少呢？Pending的时候调用的wake函数是啥意思？如果两次poll都pending我会不会wake两次？问题的根源是这个文档写了半天我还不知道啥是executor。`Each time a future is polled, it is polled as part of a "task". Tasks are the top-level futures that have been submitted to an executor.`

看完感觉还是没完全懂，还是得看一些例子模拟一下具体的执行流程，大概总结了一下目前的理解，可能有错误。

顶层的executor有很多实现，可以实现为一个队列，每次拿出来一个poll，wake函数设置为把future自己塞回队列里。所以一个future被poll以后就会跑到他返回pending或者done的位置，这个返回可以是future里面await的另一个future的返回，我理解大概是Future通过wait形成的父子关系构成一个树，每次poll根节点的顶层future就是尝试在这个树上dfs往下走一步。如果返回了pending那么wake函数会以某种方式被schedule运行，（比如socket异步，socket提供一个回调函数的接口在读完以后wake（由系统的某个eventmap实现），并且**保证Wake以后下一次pool一定会完成/进入一个不同的状态**，不然executor会退化成死循环poll等待）所以所有waker其实都是顶层executor写的，然后被下面的future传下去。总的来说，Future的poll返回阻塞的根本原因是某个底层的操作没有完成，而写在poll里的代码是肯定会执行的。

Pin：一种指针，保证指向的值不会被移动（比如指向的地址不会和另一个地址的值swap），前提是指向的类型没有实现Unpin。可以在编译期防止自引用在swap的时候爆炸。而async函数有可能生成包含内部引用的结构体，所以需要Pin。

## zCore的executor
维护一个队列，每次拿出来一个任务，如果不在sleep状态就poll一下，如果是pending就标记sleep，然后放回去。这个sleep状态其实就对应pending，wake函数的效果也就是消除这个sleep标志。