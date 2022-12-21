## 配环境
因为服务器炸了所以要重新配环境，正好记录一下过程作为文档的一部分。

### 配置zsh
``` bash
sudo apt install zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

在`.zshrc`里写`ZSH_THEME="powerlevel10k/powerlevel10k"`然后重启。

安装最重要的插件zsh-autosuggestions：`git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions`

### 安装rust
略，可能要`rustup default nightly`。

### 安装musl
``` bash
wget https://musl.cc/riscv64-linux-musl-cross.tgz
tar -xf riscv64-linux-musl-cross.tgz
```
然后直接把bin放到环境变量里就完了。

### 安装qemu
去官网下7.1.0的压缩包，然后`tar xvJf`，我也不知道tar这些东西是干啥的反正直接google就完了。之后configure然后make，把`qemu-7.1.0/build`加到环境变量里。

### 配置vscode
如果要让rust-analyzer默认使用riscv架构，需要在设置里更改cargo的target，设置里找到这项后会打开对应位置的vscode配置json，输入`"rust-analyzer.cargo.target": "riscv64gc-unknown-none-elf"`即可。

## backtrace
实现了带debuginfo的backtrace。

编译的时候把zcore的elf放到文件系统里，backtrace的时候就可以直接从这个文件读取debuginfo了。把gimli集成到了内核里进行parse。

这个过程比较曲折，一开始是想直接用gimli官方例子里的object库做parse，但是引入的时候发现object库有一个依赖memchr，这个库虽然可以no_std，但是由于cargo的一些魔法，编译的时候这个库会enable std的feature，找到了[这个issue](https://github.com/Geal/nom/issues/1457)。里面说要把cargo的version给fix到2021.但是zCore是很多个库连接在一起，有的是2018，有的是2021，也没法一个一个fix，所以就放弃了，直接换成zcore里用的xmas-elf。这个bug也是让我体会到了rust的神奇。
