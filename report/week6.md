## rust async
参照[Writing an OS in rust](https://os.phil-opp.com/async-await/#futures-in-rust)，写的很好，而且zcore也是照着这个实现的。

`We see that the first poll call starts the function and lets it run until it reaches a future that is not ready yet. If all futures on the path are ready, the function can run till the “End” state, where it returns its result wrapped in Poll::Ready. Otherwise, the state machine enters a waiting state and returns Poll::Pending. On the next poll call, the state machine then starts from the last waiting state and retries the last operation.`

所以每个async函数中可以使用await作为一条可能阻塞的边，最终形成一棵树对应函数的执行流，每条边包含一个可能需要等待的异步操作。这个异步操作有可能是很久以前就创建的，但是它只有在被await以后才会阻塞上层函数的执行。一次poll会在每个await处判断，如果异步操作已经完成就会走下去，否则才会返回pending。也就是只有函数结束或者遇到真正需要等待的异步操作阻塞的时候这次poll才会返回结果。

`The idea is that we switch to the next state as long as possible and use an explicit return Poll::Pending when we can’t continue.`

* 如果async函数里面还有async函数调用呢？

因为调用的async函数返回的也是一个future（无论那个函数多复杂或者是递归调用自己），一样的也是在await处poll状态机。调用一般函数的话就不用管了。

* 如果调用了含有async+await的一般函数呢？

编译器应该可以识别这种嵌套，可以把里面的await都拿出来形成一个大的状态机。

* 如果含有async+await的函数会递归呢？

会编译错误，需要workaround。

编译器根据生命周期为每个状态生成结构体保存这个状态中还活着的变量，以及这个状态对应的future，然后用这些结构体构成最终的顶层poll。

所以async实现了一种协作式调度，相当于有异步io等操作的时候自动yield，没有的话poll的时候可能一直跑下去。zCore里就用这个实现了线程的调度，相当于只对内核的syscall实现协作调度，用户线程的调度仍然使用时钟中断等方法。

## kprobe定位
编译的时候用nm获得符号表然后写到内核里，probe就可以直接根据函数名定位。

在zCore上试一下