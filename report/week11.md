# week11

现在基本上完全复现了前人的工作。但是我觉得有必要分享下解决这个bug的过程。

## 解决bug的过程

### 确定问题

```rust
pub fn run(&self, ctx: *const u8) -> i64 {

        if let Some(compiled_code) = &self.jited_prog {
            let result = unsafe {
                type JitedFn = unsafe fn(*const u8) -> i64;
                let f = core::mem::transmute::<*const u32, JitedFn>(compiled_code.as_ptr());
                unsafe {
            			let ptr = 0xffffffc0804df568 as *const u32;
            				error!("try deref before val= {}", *ptr);
        				}
                f(ctx)
             			unsafe {
            		let ptr = 0xffffffc0804df568 as *const u32;
            		error!("try deref after val= {}", *ptr);
        			}
            };
            return result;
        }

        todo!("eBPF interpreter missing")
    }
```

```
panicked at 'handle kernel page fault error: NOT_FOUND vaddr(0xffffffff804df568) flags(READ)', zCore/src/handler.rs:26:1


0000000000115000 0000000088080000 0000000000002000 rw-u-ad
ffffffc00c000000 000000000c000000 0000000000210000 rw---ad
ffffffc010000000 0000000010000000 0000000000009000 rw---ad
ffffffc030000000 0000000030000000 0000000010000000 rw---ad
ffffffc080200000 0000000080200000 00000000000f7000 rwx--ad
ffffffc0802f7000 00000000802f7000 0000000000030000 r----ad
ffffffc08033b000 000000008033b000 00000000001a1000 rw---ad
ffffffc0804dc000 00000000804dc000 0000000000204000 rwx--ad
ffffffc0806e1000 00000000806e1000 0000000007b1f000 rwx--ad
ffffffc088200000 0000000088200000 0000000000db5000 rw---ad
ffffffc088fb5000 0000000088fb5000 000000003604b000 rwx--ad
ffffffc0bf000000 00000000bf000000 0000000000002000 r----ad
ffffffc0bf002000 00000000bf002000 0000000000ffe000 rwx--ad
ffffffc0c0000000 00000000c0000000 0000000040000000 rwx-gad
```

经过观察可以看出，这里page fault得地址是 `0xffffffff804df568` 但是我实际的地址是 `0xffffffc0804df568` 

所以会 NOT FOUND，因为这个地址本来就不存在zCore的地址空间里！

也就是说，在编译的时候，原本的 `0xffffffc0804df568` 变成了 `0xffffffff804df568`

 ### 进一步

所以一定是提供的编译错了，而且大概率是LOAD_IMM64 有问题

最开始比较蠢，想自己一条一条的反汇编。

但是后来发现没必要这么麻烦，直接在ebpf2rv里面输出解码的结果就行了。

经过一些观察，最好发现是这里出了问题。

因为一个指令是32bit，所以load imm64需要特殊处理，我输出了 imm64这个值，发现不对。此时已经很接近了。s

```rust
// process the only 16-bytes instruction: LD_IMM_DW
        if is_load_imm64 {
            is_load_imm64 = false;
            let imm64 = (prev_imm as u64) | ((imm as u64) << 32);
            warn!("emit imm64 {:x} from {:x} | {:x} ", imm64, imm, prev_imm);
            ctx.emit_load_imm64(prev_dst, imm64 as i64);

            continue;
        }
```

继续输出log

```
emit load imm64: ffffffff8808c400 
prev_imm 8808c400 
imm ffffffc0 
prev_u64 ffffffff8808c400 
imm_u64:ffffffffffffffc0

```

可以看到，prev_imm 被 `as u64` v之后导致前32位变成了全f，这是不对的，我们expect的是 

`prev_u64 = 00000008808c400`

此时很容易猜到是发生了signed extension，看到定义 

`    *let* mut prev_imm: i32 = 0;`

所以这里直接 `prev_imm as u64` 不是我们理解的 `prev_imm as u32 as u64` 而是 `prev_imm as i64 as u64` 

在转成i64的时候符号拓展导致高位全f，所以就错了。发现bug后改起来很简单。

```rust
let imm64 = (prev_imm as u32 as u64) | ((imm as u64) << 32);
```

这个bug挺微妙的。

### 为什么rcore能跑？

![image-20221120233240741](/Users/lbr/opt/typora_image/week11//image-20221120233240741.png)

因为rcore默认地址高位全是f，所以这个错了不会对rcore有影响。

而Load imm64 在之前的测试例子中只在地址中使用，他自己出的test没有测到正好超int的部分，导致没有显现出来。

但是zCore里默认只有6个f而不是8个f，所以就暴露出来了。

## 下一步

可能需要指导下，因为也没几周了。

### 规范化系统调用

不再用 prog_load_ex 这个ad hoc的系统调用，而是按照之前叶圣给出的思路，借用某个稳定版本的libbpf，然后使用bpf_obj相关的系统调用。

libbpf 暂定用 0.8.1

linux 暂定用 5.19

### 模块化

之前的模块化还不够，引入bpf map需要os提供 copy_from_user 接口（因为单页表双页表的原因）

### 文档

希望写一个比较完善的文档。