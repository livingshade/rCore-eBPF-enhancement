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
```

可以看到，linux kernel里面并不是一个object对应一个map 或者prog的，而是很多很多。

这种实际上给模块化带来一定问题，另一方面，这个以来的rv JIT 实际上与这个是深度耦合的，虽然表面上是不同的crate，但是并不完全合理。

所以迁移时我应当进行一定的修改。另一方面，也要以“跑起来”为目的，不要改太多。

### 解析elf



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



