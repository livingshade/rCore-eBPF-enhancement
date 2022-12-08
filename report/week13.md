## async
正在执行的async函数的stacktrace可以和正常函数一样处理，不过因为async实现出现的inline比较过，需要引入debuginfo才能在内核里得到完整的调用链。

如果要跟踪已经yield的async函数就不能通过栈追踪的方式了。有几种思路：

### 改代码
给每个async函数用宏在外面套一层async函数，在poll的时候把结果汇报给全局tracer。这个是可行的，但是需要改代码。(https://crates.io/crates/async-backtrace)

### 跟踪poll函数
poll函数是很难定位的，会生成非常复杂的符号，而且生成的结果还和具体async实现有关。所以这个是不太可行的。

### 跟踪生成的闭包
首先可以从DWARF里找到async函数生成的struct：
``` rust
struct example::what::{async_fn_env#0}
	size: 3
	members:
		0[1]	__state: u8
		0[3]	<variant part>
			Unresumed: <0>
			Returned: <1>
			Panicked: <2>
			Suspend0: <3>
				0[1]	<padding>
				1[2]	__awaitee: struct example::barrr::{async_fn_env#0}
			Suspend1: <4>
				0[1]	<padding>
				1[1]	__awaitee: struct example::baz::{async_fn_env#0}
				2[1]	<padding>
```
类似这种，这个的格式是固定的，其中的awaitee就对应了子future。所以可以从debuginfo里解析出代码里future的依赖树。

对应编译器里的逻辑：
```
/// Desugar `<expr>.await` into:
/// ```ignore (pseudo-rust)
/// match ::std::future::IntoFuture::into_future(<expr>) {
///     mut __awaitee => loop {
///         match unsafe { ::std::future::Future::poll(
///             <::std::pin::Pin>::new_unchecked(&mut __awaitee),
///             ::std::future::get_context(task_context),
///         ) } {
///             ::std::task::Poll::Ready(result) => break result,
///             ::std::task::Poll::Pending => {}
///         }
///         task_context = yield ();
///     }
/// }
/// ```
```
有依赖树以后可以考虑根据闭包的执行情况跟踪。而函数生成的闭包的地址是可以找到的，跟踪这些闭包的执行情况，就可以知道async函数的执行情况。

这个闭包可能会被多次进入，不过每次的入口点应该是一样的，可以用kretprobe探测。但是这些closure的具体结构还没搞清楚，不知道能不能通过context判断返回的是done还是yield，毕竟闭包外面还是有一层poll的结构。

总的来说，这个方法应该可以实现给一个async函数子函数的闭包挂probe，但是不一定能还原出已经yield的async函数的执行情况。

当然还有一个问题是怎么在DWARF里通过闭包这个symbol找到对应的地址，不过用DWARF已经可以实现line2addr的情况下应该是可以实现的。

## 总结
如果是作为debug手段的话，async-backtrace这个库已经足够了。动态跟踪的话可以尝试用跟踪闭包的方法，由于闭包这部分的编译器代码我还没有读，只能先尝试一下挂载kprobe的效果。