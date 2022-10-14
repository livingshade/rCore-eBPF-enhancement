# 复活r eBPF 所在的rcore环境

在尝试改进之前，首先要做到复现之前的成果。

这里基于的是叶圣辉的仓库（因为上周讨论时他提出修了一些上游的bug）

 https://github.com/yesh0/rCore/blob/changelog/kernel/docs/2022F/Changelog.org

### 配置环境

```shell
# rcore/setup.sh
export ARCH=riscv64
export FILE="riscv64-linux-musl-cross"
export URL="https://musl.cc/$FILE.tgz"
if [ ! -f $FILE.tgz ]; then
    wget $URL
    tar -xzf $FILE.tgz
fi
export PATH=$PATH:$PWD/$FILE/bin%
```

```shell
# rcore/run.sh
export ARCH=riscv64
cd ./user
make all ARCH=$ARCH
make sfsimg ARCH=$ARCH
cd ../kernel
make run ARCH=$ARCH LOG=info
```

```shell
git clone --recursive https://github.com/yesh0/rCore.git rcore
rm -rf ~/.rustup
cd rcore
# add setup.sh & run.sh at pwd
./setup.sh
cd rcore/user
cd rust
rustup component add rust-src llvm-tools-preview
cd ../../kernel
rustup component add rust-src llvm-tools-preview
cd ..
./run.sh
```

这样就可以跑起来了。但是目前还有几个问题

- 一堆这种报错，需要调研

  - ```
    error: instruction requires the following: 'C' (Compressed Instructions)
      c.addi16sp sp, 32

  - 这个好像暂时不影响，先不管

- 运行某些ucore程序会page fault

  - `./ucore/hello` 
  - 这里希望的是运行eBPF-test，这个运行成功大概也表明复现的差不多了。

- 会有几个error说kprobe already exists，需要看下是什么原因



<del>暂时先这样 @10.10</del>

### page fault

和ysh以及ssf讨论了一下:

```
我试了一下暂时 badarg 看起来是没对用户指针校验, rCore page fault 有个todo，说明可能有地方没写完
然后看起来 ucore 的 sleep 实现只在 x86_64 上测试过，不兼容 riscv……内核期待一个指针它传了时间值。
不知道还有没有其它的 ABI 不兼容

```

为了确定这个是eBPF的问题还是rCore的问题，roll back到latte-C 那个版本最初的部分。

发现对于C程序仍有问题。<del>正在搞 @10.13 </del>

经过不断的	`make clean, make run`和 `debug!` ，定位到了fs对这个elf解析的问题。

与此同时，为了避免走弯路，我还联系了hzx。

hzx表示可能是因为muslgcc 工具链版本的问题，他们当时的主版本是9，而我们用的是11，rCore并没有锁这个版本。

看了一下ucore测例到ELF，可以看到有些符号在0x0，hzx表明这可能是默认链接行为的不同导致的。因为rCore并没有单独为uCore写链接脚本。

此时有两种解决方案

- 换成9版本的muslgcc，但是unfortunately musl.cc 上不能找到9版本的，这是因为 

  **2021-03-01** Release GCC 5, 6, 7, 8, 9, 10 toolchains
  \- **All previous toolchains were lost due to catastrophic disk failure. Rebuilt / rebuilding.**

  hzx表示不太能直接给我发9版本的，所以考虑手动编译，但是好像比较麻烦，暂时搁置。

- 手动写一个链接脚本 .ld ,这样就能与musl版本无关了，目前我正在研究这个方面。

@10.14



