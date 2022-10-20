## cont' 复活r eBPF 所在的rcore环境

根据叶圣的指导和石圣的指导，现在可以运行bpf_test了。

这里从头写下具体方法，改动不大。

- 首先clone这个版本，本质是叶圣的
  - 应该已经更新了里面依赖的user submodule 到 `https://github.com/yesh0/rcore-user.git`
  - 关键是 `user/ucore/CMakeList.txt` 里面 `CMAKE_C_FLAGS` 需要 `--static`

```shell
git clone --recursive git@github.com:livingshade/rCore.git
git checkout fix
cd rCore
rm -rf ~/.rustup
cd rcore
bash ./boostrap.sh
cd rcore/user
cd rust
rustup component add rust-src llvm-tools-preview
cd ../../kernel
rustup component add rust-src llvm-tools-preview
cd ..
bash ./build.sh # build user program
bash ./run.sh # run kernel
```



## 迁移并模块化eBPF

这周主要在学习，没有改什么代码。（图源latte-c展示）

![image-20221020212803404](/Users/lbr/opt/typora_image/week6//image-20221020212803404.png)



### OS无关的部分

这部分只需要会用+简单理解即可

- eBPF本身由clang得到bytecode
- 使用eBPF to rv 得到 riscv机器码
- 暂时不考虑verifier
- 参考tqx的工作https://github.com/PhlilpAlapa/OSLAB-fixbug

### OS有关部分

实现 bpf 相关的系统调用，需要仔细阅读对应代码并修改。

参考 latte-c 的rCore， 以及linux kernel doc https://www.kernel.org/doc/html/latest/bpf/instruction-set.html

- 加载bpf程序，编译（这部分不用管）
- 对于bpf map的管理
- 与对应kprobe相关的接口
  - 参考pcy的工作 http://hm1229.top/rust_ebpf_book/book/index.html

### 目标

- 迁移到zCore
  - 都是rust写的，所以应该可以直接用 eBPF2rv 和 rvjit
  - 同理，对于kprobe应该只需要较少的修改就能复用
  - 但是在async方面，可能需要考虑，这个暂时交给石圣

- 作为linux模块

  - 需要理解kernel的要求，并补全对应的接口等

    - 目前的实现中，对应规范都是兼容的。

  - 需要学习LKM

  - .....

    
