## kprobe移植
如果断点在中断处理里，那么probe函数本身也相当于中断处理函数，需要使用禁止中断的锁结构防止死锁。

[linux文档](https://www.kernel.org/doc/Documentation/locking/spinlocks.txt)讲了spinlock是否disable中断的两个版本，zCore好像直接默认使用了禁止中断的版本。rCore上实现的probe因为没考虑中断（内核态似乎直接禁用了中断？）所以直接用了普通的spinlock，需要换一下。

和OS耦合性较强的是内存分配部分，需要一些物理页作为跳板或者指令缓存。具体需求就是往一个物理页里写指令然后把PC重定向到这个页里，然后如果是uprobe的话还要把这个页map到对应的用户地址空间里。

rCore似乎是根据[blogOS的实现](https://os.phil-opp.com/paging-implementation/)把整个物理内存默认map到了虚拟内存里的一个OFFSET后面，让内核可以直接访问。zCOre的话要参考参考[Zircon 内存管理模型](http://rcore-os.cn/zCore-Tutorial/ch03-01-zircon-memory.html)。物理页以及对应虚拟地址对应一个VMO/VMAR对象，kprobe需要持有这个对象的Handle以进行交互，根据zCore内核访存的规范来就行。

还要考虑SMP的问题。

## async trace
Tokio的[async backtrace](https://tokio.rs/blog/2022-10-announcing-async-backtrace)，实现一个宏，把这个宏套在async函数上就可以追踪，

这个宏做的事情大概是把每个future外面再包一层Future，从而可以跟踪这个Future被poll的状态，然后macro里面还可以用一些操作获取closure名称等信息，追踪的方式就是用一个全局变量维护当前没完成的future。

所以在DEBUG模式下默认给所有async函数实现这个就可以了。不过根据它的代码，直接追踪对应closure看来是正确的，他就是直接用wrapper看什么时候这些closure被调用以及什么之后执行完毕（也就是poll返回Done）。

没有宏的情况下，DEBUG模式里其实可以直接找closure的call，但是什么时候结束就要看call之后的分支代码了，这个可能是可以静态分析然后probe的，他的宏就相当于强制插入了静态的跟踪点，我觉得动态插入也是有可能的。