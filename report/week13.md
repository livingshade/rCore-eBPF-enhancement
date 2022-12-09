# week13
## libbpf 

经过叶圣的指点，我学习了一下 `llvm-objdump`
的使用方法。为了能够显示重定位信息，应当使用 `llvm-objump -dr kern.o` 

这样就能正确显示了。结果如下

```c
struct bpf_map_def SEC("maps") xdp_stats_map = {
//...
}
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
```asm
Disassembly of section prog:

0000000000000000 kern_prog:
       0:	b7 01 00 00 01 00 00 00	r1 = 1
       1:	63 1a fc ff 00 00 00 00	*(u32 *)(r10 - 4) = r1
       2:	bf a2 00 00 00 00 00 00	r2 = r10
       3:	07 02 00 00 fc ff ff ff	r2 += -4
       4:	18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00	r1 = 0 ll
		0000000000000020:  R_BPF_64_64	xdp_stats_map
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
```

这里可以看到，我们只需要重定位一下 `R_BPF_64_64`
就行了。事实上，根据这个kernel patch mail [https://lore.kernel.org/lkml/1438959032-8637-16-git-send-email-acme@kernel.org/] 
ebpf 会将对map的访问变成load一个imm64作为绝对地址。而这个和之前`extern mapfd`
是一样的。`That ld_64 instruction will be recorded in relocation section.
To enable the usage of that map, relocation must be done by replacing
the immediate value by real map file descriptor so it can be found by
eBPF map functions.`

所以我们只需要类似之前的方法，将fd放进去就可以了。

事实上，ebpf对 `bpf_map_lookup_elem` 的签名是 ` void *bpf_map_lookup_elem(struct bpf_map *map, const void *key)`
而实际上，在内部实现`bpf.h`中，它的签名是 `int bpf_map_lookup_elem(int fd, const void *key, void *value)`
这也于之前将地址重定位成fd对应上了。

因此可以看出，支持这个并不需要内核的额外特殊的支持，只需要一个能够解析elf的用户态库即可。在linux中就是对应的libbpf，在用户态完成了所有的重定位工作，直接将字节码传给内核。

