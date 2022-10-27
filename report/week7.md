## rust async
从blog OS摘抄了一个最小化的async executor：
```rust
impl Task {
    pub fn new(future: impl Future<Output = ()> + 'static) -> Task {
        Task {
            future: Box::pin(future),
        }
    }

    fn poll(&mut self, context: &mut Context) -> Poll<()> {
        self.future.as_mut().poll(context)
    }
}

impl SimpleExecutor {
    pub fn run(&mut self) {
        while let Some(mut task) = self.task_queue.pop_front() {
            let waker = dummy_waker();
            let mut context = Context::from_waker(&waker);
            match task.poll(&mut context) {
                Poll::Ready(()) => {} // task done
                Poll::Pending => self.task_queue.push_back(task),
            }
        }
    }
}
```
executor在`run`的时候poll一个task的代码：
```assembly
000000000000ae50 <minimal::Task::poll> (File Offset: 0xae50):
    ae50:	48 83 ec 58          	sub    $0x58,%rsp
    ae54:	48 89 74 24 08       	mov    %rsi,0x8(%rsp)
    ae59:	48 89 7c 24 20       	mov    %rdi,0x20(%rsp)
    ae5e:	48 89 74 24 28       	mov    %rsi,0x28(%rsp)
    ae63:	48 89 7c 24 40       	mov    %rdi,0x40(%rsp)
    ae68:	e8 63 cc ff ff       	callq  7ad0 <<alloc::boxed::Box<T,A> as core::ops::deref::DerefMut>::deref_mut> (File Offset: 0x7ad0)
    ae6d:	48 89 44 24 48       	mov    %rax,0x48(%rsp)
    ae72:	48 89 54 24 50       	mov    %rdx,0x50(%rsp)
    ae77:	48 89 44 24 30       	mov    %rax,0x30(%rsp)
    ae7c:	48 89 54 24 38       	mov    %rdx,0x38(%rsp)
    ae81:	48 8b 44 24 30       	mov    0x30(%rsp),%rax
    ae86:	48 89 44 24 10       	mov    %rax,0x10(%rsp)
    ae8b:	48 8b 44 24 38       	mov    0x38(%rsp),%rax
    ae90:	48 89 44 24 18       	mov    %rax,0x18(%rsp)
    ae95:	48 8b 44 24 18       	mov    0x18(%rsp),%rax
    ae9a:	48 8b 74 24 08       	mov    0x8(%rsp),%rsi
    ae9f:	48 8b 7c 24 10       	mov    0x10(%rsp),%rdi
    aea4:	ff 50 18             	callq  *0x18(%rax)
    aea7:	88 44 24 07          	mov    %al,0x7(%rsp)
    aeab:	8a 44 24 07          	mov    0x7(%rsp),%al
    aeaf:	24 01                	and    $0x1,%al
    aeb1:	0f b6 c0             	movzbl %al,%eax
    aeb4:	48 83 c4 58          	add    $0x58,%rsp
    aeb8:	c3                   	retq   
    aeb9:	0f 1f 80 00 00 00 00 	nopl   0x0(%rax)
```
task的`poll`函数会去poll真正对应的future，可以看到调用内部future的poll在这句`callq  *0x18(%rax)`，只有运行时才知道调用的是啥。

async函数对应的代码：
```rust
#[inline(never)]
pub fn exercise() {
    let mut executor = SimpleExecutor::new();
    executor.spawn(Task::new(example(8)));
    executor.run();
}

#[inline(never)]
async fn example(t: usize) {
    let a = sy1(t).await;
    let b = sy2(t).await;
    let c = a + b;
    panic!("{}", c);
}

#[inline(never)]
async fn sy1(t: usize) -> usize {
    let a = t * 4;
    let mut r = 0;
    for i in 0..a {
        r += i;
    }
    r
}

#[inline(never)]
async fn sy2(t: usize) -> usize {
    let a = t + 2;
    let mut r = 123;
    for i in 1..a {
        r += i;
    }
    r
}

fn main() {
    exercise();
}
```

`exercise`先调用`example`获得返回的future然后传给`Task::new()`。所以`example`只是生成并返回一个future。这里可以作为函数入口插入probe，但并不是严格意义上函数开始执行的位置。
```assembly
000000000000afb0 <minimal::example> (File Offset: 0xafb0):
    afb0:	48 83 ec 48          	sub    $0x48,%rsp
    afb4:	48 89 f8             	mov    %rdi,%rax
    afb7:	48 89 44 24 08       	mov    %rax,0x8(%rsp)
    afbc:	48 89 74 24 40       	mov    %rsi,0x40(%rsp)
    afc1:	48 89 74 24 10       	mov    %rsi,0x10(%rsp)
    afc6:	c6 44 24 28 00       	movb   $0x0,0x28(%rsp)
    afcb:	48 8d 74 24 10       	lea    0x10(%rsp),%rsi
    afd0:	e8 ab cd ff ff       	callq  7d80 <core::future::from_generator> (File Offset: 0x7d80)
    afd5:	48 8b 44 24 08       	mov    0x8(%rsp),%rax
    afda:	48 83 c4 48          	add    $0x48,%rsp
    afde:	c3                   	retq   
    afdf:	90                   	nop
```
future是用`core::future::from_generator`生成的。每个async函数都会编译出一个对应的生成函数，但是这里是找不到具体的future信息的。

生成的future的poll函数是可以找到的，符号为`<<core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll>`，每个async函数都有一个对应的poll。

以顶层Future`example`为例，他的主体是一个closure`minimal::example::{{closure}}`，应该是为了适应generator编译器生成的闭包？

这个闭包会在某个poll里被调用，因为是顶层future通过executor进行poll，在汇编里是看不出来被调用的，运行时才能体现。

看一下这个闭包里的call：
```assembly
callq  afe0 <minimal::sy1> (File Offset: 0xafe0)
callq  7e20 <<F as core::future::into_future::IntoFuture>::into_future> (File Offset: 0x7e20)
callq  be00 <core::future::get_context> (File Offset: 0xbe00)
callq  7e60 <<core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll> (File Offset: 0x7e60)

b010 <minimal::sy2> (File Offset: 0xb010)
callq  7e40 <<F as core::future::into_future::IntoFuture>::into_future> (File Offset: 0x7e40)
callq  be00 <core::future::get_context> (File Offset: 0xbe00)
callq  7fb0 <<core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll> (File Offset: 0x7fb0)
```
分成了`sy1`和`sy2`两个部分，分析一下`sy1(t).await`这句话的执行流程。

先调用对应函数获取Genfuture，然后调用`into_future`。`into_future`是.await编译后的结果，[文档](https://doc.rust-lang.org/std/future/trait.IntoFuture.html)：

`The .await keyword desugars into a call to IntoFuture::into_future first before polling the future to completion. IntoFuture is implemented for all T: Future which means the into_future method will be available on all futures.`

但是还是不知道`into_future`是干嘛的。似乎Future在底层是变成Generator实现的，应该和这个有关，要研究的话可能要了解一下Genfuture是啥。

之后用`get_context`获取poll用的context然后传给poll。poll是一个`<<core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll>`，每个future对应的poll函数都不一样（虽然标志一样）。

第一次调用`7e60`处的poll对应`sy1`函数的poll，里面有`callq  b840 <minimal::sy1::{{closure}}>`，这就对应了`sy1`的执行过程。返回以后根据结果判断是否继续poll`sy2`。

总结下来，async函数调用的流程在汇编中体现为：

1. 每个async函数被转换为一个返回future的函数以及一个对应函数本身执行流程的closure。
2. 函数被调用，通过`core::future::from_generator`返回一个Genfuture，然后调用into_future。
3. 通过`core::future::get_context`获取poll的参数。
4. 调用`<<core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll>`尝试poll。这里是把前面获得的Genfuture变成了future，然后调用poll。
5. 根据poll的结果判断是否继续poll后面的future，对应状态机的转移。

推测这个closure是传入状态机的结构体里执行的，每次poll会根据当前状态判断执行哪个部分。目前看来跟踪函数的话只需要对这个closure操作就行了。

以上分析都是基于unoptimized + debuginfo的版本，release模式下closure都消失了，似乎直接被合并到了poll里，poll里很多call变成了_GLOBAL_OFFSET_TABLE_寻址。