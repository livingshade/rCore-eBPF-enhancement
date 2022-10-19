### cont' 复活r eBPF 所在的rcore环境

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



