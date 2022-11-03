## kprobe移植
如果断点在中断处理里，那么probe函数本身也相当于中断处理函数，需要使用禁止中断的锁结构防止死锁。

[linux文档](https://www.kernel.org/doc/Documentation/locking/spinlocks.txt)讲了spinlock是否disable中断的两个版本，zCore好像直接默认使用了禁止中断的版本。rCore上实现的probe因为没考虑中断（内核态直接禁用了中断？）所以直接用了普通的spinlock，需要换一下。

和OS耦合性较强的是内存分配部分，需要一些物理页作为跳板或者指令缓存。具体需求就是往一个物理页里写指令然后把PC重定向到这个页里，然后如果是uprobe的话还要把这个页map到对应的用户地址空间里。

rCore似乎是根据[blogOS的实现](https://os.phil-opp.com/paging-implementation/)把整个物理内存默认map到了虚拟内存里的一个OFFSET后面，让内核可以直接访问。zCOre的话要参考参考[Zircon 内存管理模型](http://rcore-os.cn/zCore-Tutorial/ch03-01-zircon-memory.html)。物理页以及对应虚拟地址对应一个VMO/VMAR对象，kprobe需要持有这个对象的Handle以进行交互。

具体实现：新建一个Vmo给kprobe持有，commit申请物理帧作为指令缓存区，应该可以RAII。zCore好像也是直接用OFFSET实现内核访存，直接用kernel-hal里的接口就行。

```
error: instruction requires the following: 'C' (Compressed Instructions)
    c.addi16sp sp, 32
```
这个报错之前rCore就有，现在发现是kprobe的test汇编里面写的，应该是新版本rust编译器的`global_asm!`特性变了，暂时不管。

写好以后遇到WRITE的page fault。用qemu看页表以后发现zCore的内核代码段是没有write权限的。于是在`kernel-hal`层改了`init_kernel_page_table()`里设置的权限。这样kprobe的setup就能完成了。

最后需要在断点的时候调用kprobe的处理函数。这里遇到了一个问题：因为`trap_handler`在`kernal-hal`里，`kprobe`实现在`zircon-object`里，如果`kernel-hal`要调用这个handler就要引入整个`zircon-object`作为crate，就有了循环引用的问题。

初步的解决方案是把`kprobe`的handler作为一个`extern "C"`函数放在`kernel-hal`里，然后在`kprobe`里实现这个函数。否则好像只能把`probe`独立出来作为一个crate放在最上层，这里怎么设计还要考虑一下。

之后的问题就是指令buffer在data段不可以执行，所以又改了这一段的权限。虽然vmo在commit的时候设置了execute权限，但是可能因为commit获得的帧已经被map过了，权限并没有被更改，后面要找一下正确的实现方法。

现在可以正确执行`kprobes`的测试，以info方式输出。因为环境变量不管用所以直接在main里硬编码了log级别为info。

## async trace
Tokio的[async backtrace](https://tokio.rs/blog/2022-10-announcing-async-backtrace)，实现一个宏，把这个宏套在async函数上就可以追踪，

这个宏做的事情大概是把每个future外面再包一层Future，从而可以跟踪这个Future被poll的状态，然后macro里面还可以用一些操作获取closure名称等信息，追踪的方式就是用一个全局变量维护当前没完成的future。

所以在DEBUG模式下默认给所有async函数实现这个就可以了。不过根据它的代码，直接追踪对应closure看来是正确的，他就是直接用wrapper看什么时候这些closure被调用以及什么之后执行完毕（也就是poll返回Done）。

没有宏的情况下，DEBUG模式里其实可以直接找closure的call，但是什么时候结束就要看call之后的分支代码了，这个可能是可以静态分析然后probe的，他的宏就相当于强制插入了静态的跟踪点，我觉得动态插入也是有可能的。
