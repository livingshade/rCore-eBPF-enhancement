## 现有的不足之处

对于 eBPF object并不完全，但是eBPF本身需要对 elf 进行很多的解析工作，这方面需要拓展。而且最关键的是，之前的做法是手动 `bpf_prog_load_ex` 来加载.o的，其中需要手动传一个 `*struct* bpf_map_fd_entry` 。而这些并不是linux的标准，这些系统调用也不存在。

```C
void test_bpf_prog() {
    int map_fd = create_map();

    struct stat stat;
    int fd = open("./map.o", O_RDONLY);
    if (fd < 0) {
        cprintf("open file failed!\n");
        return;
    }
    unsigned *p = (unsigned *) ret;
    read(fd, p, prog_size);
    cprintf("ELF content: %x %x %x %x\n", p[0], p[1], p[2], p[3]);

    struct bpf_map_fd_entry map_array[] = {
        { .name = "map_fd", .fd = map_fd },
    };
    int bpf_fd = bpf_prog_load_ex(p, prog_size, map_array, 1);
    cprintf("load ex: %x\n", bpf_fd);

    const char *target = "kprobe:<rcore::syscall::Syscall>::sys_fork";
    cprintf("attach: %d\n", bpf_prog_attach(target, bpf_fd));
    close(fd);
}
```

对比一下C中 `bpf_object` 的定义

```C
// latte-c
union bpf_attr {
    struct {
        uint32_t map_type;
        uint32_t key_size;
        uint32_t value_size;
        uint32_t max_entries;
        // more...
    };

    struct {
        uint32_t map_fd;
        uint64_t key;
        union {
            uint64_t value;
            uint64_t next_key;
        };
        uint64_t flags;
    };

    // for custom BPF_PROG_LOAD_EX
    struct {
        uint64_t prog;
        uint32_t prog_size;
		uint32_t map_array_len;
		struct bpf_map_fd_entry *map_array;
    };

	// for custom BPF_PROG_ATTACH
	struct {
		const char *target;
		uint32_t prog_fd;
	};
};

// linux
struct bpf_object {
	char name[BPF_OBJ_NAME_LEN];
	char license[64];
	__u32 kern_version;

	struct bpf_program *programs;
	size_t nr_programs;
	struct bpf_map *maps;
	size_t nr_maps;
	size_t maps_cap;
	struct bpf_secdata sections;

	bool loaded;
	bool has_pseudo_calls;
	bool relaxed_core_relocs;

	/*
	 * Information when doing elf related work. Only valid if fd
	 * is valid.
	 */
	struct {
		int fd;
		const void *obj_buf;
		size_t obj_buf_sz;
		Elf *elf;
		GElf_Ehdr ehdr;
		Elf_Data *symbols;
		Elf_Data *data;
		Elf_Data *rodata;
		Elf_Data *bss;
		size_t strtabidx;
		struct {
			GElf_Shdr shdr;
			Elf_Data *data;
		} *reloc;
		int nr_reloc;
		int maps_shndx;
		int btf_maps_shndx;
		int text_shndx;
		int data_shndx;
		int rodata_shndx;
		int bss_shndx;
	} efile;
	/*
	 * All loaded bpf_object is linked in a list, which is
	 * hidden to caller. bpf_objects__<func> handlers deal with
	 * all objects.
	 */
	struct list_head list;

	struct btf *btf;
	struct btf_ext *btf_ext;

	void *priv;
	bpf_object_clear_priv_t clear_priv;

	struct bpf_capabilities caps;

	char path[];
};

struct bpf_program {
        char *name;
        char *sec_name;
        size_t sec_idx;
        const struct bpf_sec_def *sec_def;
        /* this program's instruction offset (in number of instructions)
         * within its containing ELF section
         */
        size_t sec_insn_off;
        /* number of original instructions in ELF section belonging to this
         * program, not taking into account subprogram instructions possible
         * appended later during relocation
         */
			  size_t sec_insn_cnt;
        /* Offset (in number of instructions) of the start of instruction
         * belonging to this BPF program  within its containing main BPF
         * program. For the entry-point (main) BPF program, this is always
         * zero. For a sub-program, this gets reset before each of main BPF
         * programs are processed and relocated and is used to determined
         * whether sub-program was already appended to the main program, and
         * if yes, at which instruction offset.
         */
        size_t sub_insn_off;

        /* instructions that belong to BPF program; insns[0] is located at
         * sec_insn_off instruction within its ELF section in ELF file, so
         * when mapping ELF file instruction index to the local instruction,
         * one needs to subtract sec_insn_off; and vice versa.
         */
        struct bpf_insn *insns;
  
        /* actual number of instruction in this BPF program's image; for
         * entry-point BPF programs this includes the size of main program
         * itself plus all the used sub-programs, appended at the end
         */
  			size_t insns_cnt;
        struct reloc_desc *reloc_desc;
        int nr_reloc;

        /* BPF verifier log settings */
        char *log_buf;
        size_t log_size;
        __u32 log_level;
        struct bpf_object *obj;

        int fd;
        bool autoload;
        bool autoattach;
        bool mark_btf_static;
        enum bpf_prog_type type;
        enum bpf_attach_type expected_attach_type;
          int prog_ifindex;
        __u32 attach_btf_obj_fd;
        __u32 attach_btf_id;
        __u32 attach_prog_fd;

        void *func_info;
        __u32 func_info_rec_size;
        __u32 func_info_cnt;

        void *line_info;
        __u32 line_info_rec_size;
        __u32 line_info_cnt;
        __u32 prog_flags;
};

struct bpf_map {
        struct bpf_object *obj;
        char *name;
        /* real_name is defined for special internal maps (.rodata*,
         * .data*, .bss, .kconfig) and preserves their original ELF section
         * name. This is important to be be able to find corresponding BTF
         * DATASEC information.
         */
        char *real_name;
        int fd;
        int sec_idx;
        size_t sec_offset;
        int map_ifindex;
        int inner_map_fd;
        struct bpf_map_def def;
                __u32 numa_node;
        __u32 btf_var_idx;
        __u32 btf_key_type_id;
        __u32 btf_value_type_id;
        __u32 btf_vmlinux_value_type_id;
        enum libbpf_map_type libbpf_type;
        void *mmaped;
        struct bpf_struct_ops *st_ops;
        struct bpf_map *inner_map;
        void **init_slots;
        int init_slots_sz;
        char *pin_path;
        bool pinned;
        bool reused;
        bool autocreate;
        __u64 map_extra;
};

```

可以看到，linux kernel里面并不是一个object对应一个map 或者prog的，而是很多很多。

这种实际上给模块化带来一定问题，另一方面，这个以来的rv JIT 实际上与这个是深度耦合的，虽然表面上是不同的crate，但是并不完全合理。

这里目前的做法就是抽象出一个表示elf的object，但是底下还是以prog或者map作为主体。



### 和kprobe链接

目前的实现是这样的

```rust
fn bpf_prog_attach(target: str, fd: i32) {
    addr = resolve_symbol(target)
  	ctx = kprobe_context::from(addr)
    prog = find_prog_by_fd(fd)
    handler = bpf_generate_handler(prog, ctx)
    kprobe = KProbe{
      paddr = addr,
      handler = handler,
  }
  // note that there can be multiple prog attach to one tracepoint
  // a vector of prog is needed, but we omit that
    sys_register_kprobe(kprobe);
}


```

这里和kprobe有关。这里问题在于trapframe的移植，目前使用的是TrapFrame 0.9

## 移植zCore

https://github.com/cubele/zCore

一边改一边学习zCore。

根据杨德睿工程师的指导，大概结构是这样的

```
				 <--- linux_syscall (i.e. memory)
z_object <--- linux_object  <--- linux_syscall (i.e. ipc)
         <--- z_syscall
```

所以我们的实现都是应该放在 z_object 里，然后看情况是否需要添加 linux_object 来实现对应的syscall

目前还在写。

## 远期目标

能够做到类似这样的效果

```C
// lathist_user.c
int main(int argc, char **argv)
{
	struct bpf_link *links[2];
	struct bpf_program *prog;
	struct bpf_object *obj;
	char filename[256];
	int map_fd, i = 0;

	snprintf(filename, sizeof(filename), "%s_kern.o", argv[0]);
	obj = bpf_object__open_file(filename, NULL);

	/* load BPF program */
   bpf_object__load(obj)
     
	map_fd = bpf_object__find_map_fd_by_name(obj, "my_lat");
	
cleanup:
	for (i--; i >= 0; i--)
		bpf_link__destroy(links[i]);

	bpf_object__close(obj);
	return 0;
}

// lathist_kern.c

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__type(key, int);
	__type(value, u64);
	__uint(max_entries, MAX_CPU);
} my_map SEC(".maps");

SEC("kprobe/trace_preempt_off")
int bpf_prog1(struct pt_regs *ctx)
{
	int cpu = bpf_get_smp_processor_id();
	u64 *ts = bpf_map_lookup_elem(&my_map, &cpu);

	if (ts)
		*ts = bpf_ktime_get_ns();

	return 0;
}

struct {
	__uint(type, BPF_MAP_TYPE_ARRAY);
	__type(key, int);
	__type(value, long);
	__uint(max_entries, MAX_CPU * MAX_ENTRIES);
} my_lat SEC(".maps");

SEC("kprobe/trace_preempt_on")
int bpf_prog2(struct pt_regs *ctx)
{
	u64 *ts, cur_ts, delta;
	int key, cpu;
	long *val;

	cpu = bpf_get_smp_processor_id();
	ts = bpf_map_lookup_elem(&my_map, &cpu);
  
	cur_ts = bpf_ktime_get_ns();
	delta = log2l(cur_ts - *ts);
	key = cpu * MAX_ENTRIES + delta;
	val = bpf_map_lookup_elem(&my_lat, &key);
	if (val)
		__sync_fetch_and_add((long *)val, 1);

	return 0;

}

```



