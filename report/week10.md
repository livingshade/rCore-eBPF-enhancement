# week 10

### 补全之前的一些测试

- kretprobe
- get time 
- get processor id ...

## bpf map

- 完成了helper function， 从用户态都能正常执行
  - 测试了
- 但是对于加载到内核中执行程序，还有些问题

```c++
#include "bpf.h"

extern int map_fd;

int foo()
{
  	bpf_printk(map_fd);
    // refer to map_fd
    return 1;
}

```

这个foo是要被加载到内核中的，这里的问题规约到了读map fd的时候会page fault：NOT FOUND

目前发现page fault 应该是因为使用了 extern int，这个应该是他们在之前实现的一个ad hoc方案，要手动传一个结构体把map_fd的地址写好。

然后在bpf里面需要读这个内核地址，而地址计算的时候用的是 LOAD_IMM64 这里出了问题。

目前这个bug还没修好，大概看了一下加载的程序是没问题的，qemu看页表也是有的，不应该是not found。所以现在很费解是什么原因，还需要再debug

