## rust async
参照[Writing an OS in rust](https://os.phil-opp.com/async-await/#futures-in-rust)，写的很好，而且zcore也是照着这个实现的。

`We see that the first poll call starts the function and lets it run until it reaches a future that is not ready yet. If all futures on the path are ready, the function can run till the “End” state, where it returns its result wrapped in Poll::Ready. Otherwise, the state machine enters a waiting state and returns Poll::Pending. On the next poll call, the state machine then starts from the last waiting state and retries the last operation.`

所以每个async函数中可以使用await作为一条可能阻塞的边，最终形成一棵树对应函数的执行流，每条边包含一个可能需要等待的异步操作。这个异步操作有可能是很久以前就创建的，但是它只有在被await以后才会阻塞上层函数的执行。一次poll会在每个await处判断，如果异步操作已经完成就会走下去，否则才会返回pending。也就是只有函数结束或者遇到真正需要等待的异步操作阻塞的时候这次poll才会返回结果。

`The idea is that we switch to the next state as long as possible and use an explicit return Poll::Pending when we can’t continue.`

* 如果async函数里面还有async函数调用呢？

因为调用的async函数返回的也是一个future，一样的也是在await处poll状态机。调用一般函数的话就不用管了。

* 如果含有async+await的函数会递归呢？

似乎直接写会编译错误，需要workaround。

编译器根据生命周期为每个状态生成结构体保存这个状态中还活着的变量，以及这个状态对应的future，然后用这些结构体构成最终的顶层poll。

所以async实现了一种协作式调度，相当于有异步io等操作的时候自动yield，没有的话poll的时候可能一直跑下去。zCore里就用这个实现了线程的调度，相当于只对内核的syscall实现协作调度，用户线程的调度仍然使用时钟中断等方法。

## kprobe定位
如果内核里有自己的symbol table，probe就可以直接根据函数名定位。rCore里的`fill.sh`实现了nm以后用rustfilt获取unmangled的符号，排序并压缩后打包写到内核里的某个位置，rCore运行的时候内核的lkm里面的`ModuleManager`会把这个位置的压缩包解压获得symbol table。

把他的sh重构成了python脚本，现在的nm支持demangle rust的符号以及根据地址排序，不需要rustfilt以及手动sort了。

开了个playground试一试async函数。因为async函数会被变成返回Future的函数，所以函数本体的入口还是可以找到的，可以实现入口处插入probe，但是没法进一步跟踪。函数里会调用一个`<core::future::from_generator>`获取对应的Future，每个async函数都有一个专门的函数。

想要跟踪函数结束的话需要看他poll的结构，目前找到了一些`<<core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll::{{closure}}>`，计划从poll开始往上追溯看看能不能把future和poll对应起来。因为现在的例子用的是futures库的executor，追溯比较麻烦，应该要自己实现一个简单的executor看一下。