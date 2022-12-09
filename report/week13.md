## 文档
正在把kprobe和async的一些文档搬运到在https://cubele.github.io/probe-docs/里，预计有之前kprobe文档的完善和async rust的一些内容。

## async
尝试了一下在不修改代码的前提下跟踪pending的async函数，但是失败了。原因是poll函数的符号找不到，想通过closure的执行跟踪但是发现有一些closure的地址找不到，并且根据Future实现的不一样甚至可能找不到对应的closure。还是套一层Profiler靠谱。如果想只依赖debuginfo做的话可能要进一步编译器支持，[这个issue](https://github.com/rust-lang/rust/issues/73522)在讨论，还是open的。

中间做了一个用debuginfo输出async函数依赖图的工具在https://github.com/cubele/rust-async-tree-parser。

## 计划
后面就完善一下kprobe的文档以及代码注释（本来几乎没有注释），然后重点要写清楚移植zCore的时候一些adhoc的做法。
