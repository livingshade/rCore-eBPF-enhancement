# 复活r eBPF 所在的rcore环境

由于各种原因，需要复活一下。这里基于的是叶圣的仓库 https://github.com/yesh0/rCore/blob/changelog/kernel/docs/2022F/Changelog.org

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

- 运行任意一个ucore程序会page fault

  - `./ucore/hello` 

- 会有几个error说kprobe already exists，需要看下是什么原因



暂时先这样 @10.10