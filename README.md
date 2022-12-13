# Portable kprobe and ebpf module

## Usage

1. add dependencies in Cargo.toml to your project.
2. change `osutils.rs` according to your OS.
3. If your OS dosen't use the trapframe crate, modify trapframe related functions in arch/{arch}/trapframe.rs accordingly.
4. call `kprobes_breakpoint_handler` with trapframe in your OS's breakpoint handler.

## example

port on zCore: https://github.com/cubele/zCore