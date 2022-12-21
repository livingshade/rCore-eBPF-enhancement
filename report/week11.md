## DWARF parser
### gimli
[gimli](https://github.com/gimli-rs)库用rust实现了解析DWARF的功能，基于此实现了addr2line等工具，支持no_std。

[ddbug](https://github.com/gimli-rs/ddbug)可以格式化输出DWARF的信息。

用这个工具可以看到一个async函数的状态机，以zCore的syscall为例：
``` rust
struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}
	size: 1352
	members:
		0[1352]	<variant part>
			Unresumed: <0>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	args: [usize; 6]
				56[48]	<padding>
				104[4]	num: u32
				108[1244]	<padding>
			Returned: <1>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	args: [usize; 6]
				56[48]	<padding>
				104[4]	num: u32
				108[1244]	<padding>
			Panicked: <2>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	args: [usize; 6]
				56[48]	<padding>
				104[4]	num: u32
				108[1244]	<padding>
			Suspend0: <3>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[112]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_read::{async_fn_env#0}>
				232[1120]	<padding>
			Suspend1: <4>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[128]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_pread::{async_fn_env#0}>
				248[1104]	<padding>
			Suspend2: <5>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[136]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_readv::{async_fn_env#0}>
				256[1096]	<padding>
			Suspend3: <6>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[1232]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_sendfile::{async_fn_env#0}>
			Suspend4: <7>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[1192]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_copy_file_range::{async_fn_env#0}>
				1312[40]	<padding>
			Suspend5: <8>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[304]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::poll::{impl#0}::sys_pselect6::{async_fn_env#0}>
				424[928]	<padding>
			Suspend6: <9>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[168]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::poll::{impl#0}::sys_ppoll::{async_fn_env#0}>
				288[1064]	<padding>
			Suspend7: <10>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[56]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::vm::{impl#0}::sys_mmap::{async_fn_env#0}>
				176[1176]	<padding>
			Suspend8: <11>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[112]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::net::{impl#0}::sys_connect::{async_fn_env#0}>
				232[1120]	<padding>
			Suspend9: <12>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[120]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::net::{impl#0}::sys_accept::{async_fn_env#0}>
				240[1112]	<padding>
			Suspend10: <13>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[184]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::net::{impl#0}::sys_recvfrom::{async_fn_env#0}>
				304[1048]	<padding>
			Suspend11: <14>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[216]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::net::{impl#0}::sys_recvmsg::{async_fn_env#0}>
				336[1016]	<padding>
			Suspend12: <15>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[176]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::task::{impl#0}::sys_wait4::{async_fn_env#0}>
				296[1056]	<padding>
			Suspend13: <16>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[296]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::misc::{impl#0}::sys_futex::{async_fn_env#0}>
				416[936]	<padding>
			Suspend14: <17>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[72]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::task::{impl#0}::sys_nanosleep::{async_fn_env#0}>
				192[1160]	<padding>
			Suspend15: <18>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[136]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::time::{impl#0}::sys_clock_nanosleep::{async_fn_env#0}>
				256[1096]	<padding>
			Suspend16: <19>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[120]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::ipc::{impl#0}::sys_semop::{async_fn_env#0}>
				240[1112]	<padding>
			Suspend17: <20>
				0[8]	self: &mut linux_syscall::Syscall
				8[48]	<padding>
				56[48]	args: [usize; 6]
				56[48]	args: [usize; 6]
				104[4]	<padding>
				108[4]	num: u32
				108[4]	num: u32
				112[8]	<padding>
				120[64]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::{impl#0}::riscv64_syscall::{async_fn_env#0}>
				184[1168]	<padding>
		112[1]	__state: u8
		113[1239]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Unresumed
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	args: [usize; 6]
		56[48]	<padding>
		104[4]	num: u32
		108[1244]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Returned
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	args: [usize; 6]
		56[48]	<padding>
		104[4]	num: u32
		108[1244]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Panicked
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	args: [usize; 6]
		56[48]	<padding>
		104[4]	num: u32
		108[1244]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend0
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[112]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_read::{async_fn_env#0}>
		232[1120]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend1
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[128]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_pread::{async_fn_env#0}>
		248[1104]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend2
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[136]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_readv::{async_fn_env#0}>
		256[1096]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend3
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[1232]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_sendfile::{async_fn_env#0}>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend4
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[1192]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::file::{impl#0}::sys_copy_file_range::{async_fn_env#0}>
		1312[40]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend5
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[304]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::poll::{impl#0}::sys_pselect6::{async_fn_env#0}>
		424[928]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend6
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[168]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::file::poll::{impl#0}::sys_ppoll::{async_fn_env#0}>
		288[1064]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend7
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[56]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::vm::{impl#0}::sys_mmap::{async_fn_env#0}>
		176[1176]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend8
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[112]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::net::{impl#0}::sys_connect::{async_fn_env#0}>
		232[1120]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend9
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[120]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::net::{impl#0}::sys_accept::{async_fn_env#0}>
		240[1112]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend10
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[184]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::net::{impl#0}::sys_recvfrom::{async_fn_env#0}>
		304[1048]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend11
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[216]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::net::{impl#0}::sys_recvmsg::{async_fn_env#0}>
		336[1016]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend12
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[176]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::task::{impl#0}::sys_wait4::{async_fn_env#0}>
		296[1056]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend13
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[296]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::misc::{impl#0}::sys_futex::{async_fn_env#0}>
		416[936]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend14
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[72]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::task::{impl#0}::sys_nanosleep::{async_fn_env#0}>
		192[1160]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend15
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[136]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::time::{impl#0}::sys_clock_nanosleep::{async_fn_env#0}>
		256[1096]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend16
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[120]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::ipc::{impl#0}::sys_semop::{async_fn_env#0}>
		240[1112]	<padding>

struct linux_syscall::{impl#0}::syscall::{async_fn_env#0}::Suspend17
	size: 1352
	members:
		0[8]	self: &mut linux_syscall::Syscall
		8[48]	<padding>
		56[48]	args: [usize; 6]
		56[48]	args: [usize; 6]
		104[4]	<padding>
		108[4]	num: u32
		108[4]	num: u32
		112[8]	<padding>
		120[64]	__awaitee: struct core::future::from_generator::GenFuture<linux_syscall::{impl#0}::riscv64_syscall::{async_fn_env#0}>
		184[1168]	<padding>
```
https://swatinem.de/blog/async-codegen/
