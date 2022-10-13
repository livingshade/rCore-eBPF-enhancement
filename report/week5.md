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

发现对于C程序仍有问题。正在搞 @10.13



