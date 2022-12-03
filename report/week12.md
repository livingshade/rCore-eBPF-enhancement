# week 12
本周主要在造机+复习网安考试，没有做太多东西。
主要是探索了一些libbpf的可能性。

## libbpf

尝试解析libbpf的内容。只是作为初步尝试。 

kernel version: 5.4.0-132-generic

首先需要clone libbpf，然后需要按包括clang，libelf等一些依赖。

然后尝试 `clang -target bpf -c kern.c -o kern.o` 会发现提示 `<asm/types.h>` 不存在。以及一些依赖错误。

这里实际上需要按一些kernel headers，或者自己配一下依赖。比较好的办法是根据xdp-tutorial的教程，这样基本上不回缺依赖了。

https://github.com/xdp-project/xdp-tutorial/blob/master/setup_dependencies.org

遇到问题也可以去bcc或者xdp这种地方的issue里面搜一搜，也许就能找到答案。	

```c
struct bpf_map_def SEC("maps") xdp_stats_map = {
    .type = BPF_MAP_TYPE_ARRAY,
    .key_size = sizeof(__u32),
    .value_size = sizeof(__u32),
    .max_entries = 256,
};

SEC("prog")
int kern_prog(struct xdp_md *ctx) {

  __u32 key = 1;
  int *rec = 0;
  rec = bpf_map_lookup_elem(&xdp_stats_map, &key);
  if (!rec)
    return -1;
  ++*rec;
  return 0;
}
```

这样的代码编译出.o 后再objdump 出来后是这样的结果
```assembly

Disassembly of section prog:


0000000000000000 kern_prog:
       0:	b7 01 00 00 00 00 00 00	r1 = 0
       1:	63 1a fc ff 00 00 00 00	*(u32 *)(r10 - 4) = r1
       2:	bf a2 00 00 00 00 00 00	r2 = r10
       3:	07 02 00 00 fc ff ff ff	r2 += -4
       4:	18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00	r1 = 0 ll
       6:	85 00 00 00 01 00 00 00	call 1
       7:	18 01 00 00 ff ff ff ff 00 00 00 00 00 00 00 00	r1 = 4294967295 ll
       9:	15 00 04 00 00 00 00 00	if r0 == 0 goto +4 <LBB0_2>
      10:	61 01 00 00 00 00 00 00	r1 = *(u32 *)(r0 + 0)
      11:	07 01 00 00 01 00 00 00	r1 += 1
      12:	63 10 00 00 00 00 00 00	*(u32 *)(r0 + 0) = r1
      13:	b7 01 00 00 00 00 00 00	r1 = 0

0000000000000070 LBB0_2:
      14:	bf 10 00 00 00 00 00 00	r0 = r1
      15:	95 00 00 00 00 00 00 00	exit

Disassembly of section .relprog:

0000000000000000 .relprog:
       0:	20 00 00 00 00 00 00 00	r0 = *(u32 *)skb[0]
       1:	01 00 00 00 25 00 00 00	<unknown>

Disassembly of section maps:

0000000000000000 xdp_stats_map:
       0:	02 00 00 00 04 00 00 00	<unknown>
       1:	04 00 00 00 00 01 00 00	w0 += 256
       2:	00	<unknown>
       2:	00	<unknown>
       2:	00	<unknown>
       2:	00	<unknown>
```

https://docs.kernel.org/bpf/llvm_reloc.html

根据叶圣的指导，这一部分实际上就是一个结构体的数据，如下

```c
typedef struct
{
  Elf64_Addr    r_offset;  // Offset from the beginning of section.
  Elf64_Xword   r_info;    // Relocation type and symbol index.
} Elf64_Rel;
```

那这样的话，实际上就可以进行重定向。


