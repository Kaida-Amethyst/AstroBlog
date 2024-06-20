---
title: 在Ubuntu系统上安装Clang编译器
published: 2023-11-27
description: 记录一下在Ubuntu系统上安装clang编译器的方法以及注意事项。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Linux/llvm.png
category: 开发工具
tags: [Linux, Ubuntu]
---

---------

## 直接通过包管理器安装

使用下面的命令。

```bash
sudo apt install clang clang++
```

但直接通过包管理器安装，可能安装的clang和clang++编译器的版本不够，如果希望安装更新版的编译器，可能需要其它的办法。

## 使用LLVM存储库脚本安装clang

首先从LLVM存储库下载安装脚本，会直接下载最新的stable版本的llvm和clang，注意为脚本添加执行权限。

```bash
wget https://apt.llvm.org/llvm.sh
chmod +x llvm.sh
sudo ./llvm.sh
```

如果需要安装特定的的版本，可以指定版本号，例如如果要安装clang-19（本博客写成时，clang-19还是unstable的版本），可以使用下面的命令

```bash
sudo ./llvm.sh 19
```

## 通过源码安装clang

有的时候因为各种原因，我们可能希望直接通过源码来进行安装，这里介绍一下通过源码安装clang的流程，以下是详细步骤：

### 安装依赖

在开始之前，你需要确保你的系统安装了构建clang所需的依赖。使用以下命令安装：

```bash
sudo apt update
sudo apt install build-essential cmake ninja-build git python3
```

### 下载LLVM和Clang源码

首先，从LLVM的官方仓库克隆源码：

```bash
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
```

（Optional）你可以选择特定的分支或标签，例如，选择版本16.0.0：

```bash
git checkout llvmorg-16.0.0
```

如果直接就想安装最新版的llvm，也可以直接使用`--depth 1`，可以节省不少硬盘空间。

```bash
git clone --depth 1 https://github.com/llvm/llvm-project.git
```

### 创建构建目录

接下来，创建一个独立的构建目录，并进入该目录：

```bash
mkdir build
cd build
```

### 配置和生成构建文件

使用CMake配置构建文件。这里使用Ninja作为构建工具，你也可以选择其他工具如Make：

```bash
cmake -G Ninja ../llvm -DLLVM_ENABLE_PROJECTS=clang -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD=X86
```

### 编译和安装

运行以下命令开始编译：

```bash
ninja
```

编译过程可能需要一些时间，具体取决于你的硬件配置。编译完成后，安装clang：

```bash
sudo ninja install
```

### 配置环境变量

为了方便使用，你可以将clang添加到系统路径中。编辑`~/.bashrc`或`~/.zshrc`文件：

```bash
echo "export PATH=/usr/local/bin:$PATH" >> ~/.bashrc
source ~/.bashrc
```

至此，你已经成功通过源码安装了clang编译器。

## 验证安装

你可以通过以下命令验证安装是否成功：

```bash
clang --version
clang++ --version
```

如果显示正确的版本信息，说明安装已经成功。

但要注意的是，因为安装方法的不同，直接使用clang或者clang++，系统可能会提示没有这样的工具。这种情况下，首先尝试使用clang带版本号的命令，例如，如果先前通过指定版本号安装的是clang 19，那么你可能需要尝试：

```bash
clang-19 --version
clang++-19 --version
```

如果还是找不到，则可以在`/usr/bin`，`/usr/local/bin`和`~/.local/bin`的目录下，去寻找是否有包含clang名字的可执行程序，如果都没有再考虑重新安装。

## 配置clang版本

如果系统上有多个clang编译器（或者clang命令不能正常使用的话），可以做一个软链接将clang连接到特定的clang版本，或者也可以通过`update-alternatives`命令直接管理。

本博客写成的时候安装的是clang-18，因此下面的命令使用的是`clang++-18`和`clang-18`。

```bash
sudo update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-18 100
sudo update-alternatives --install /usr/bin/clang clang /usr/bin/clang-18 100
```

## libc++

通过包管理器或者LLVM存储库安装的clang编译器，是默认不安装头文件和库文件的，如果机器上本身有g++，clang会直接使用libstdc++，如果需要使用llvm的libc++，就需要额外安装，使用下面的命令：

```cpp
sudo apt-get install libc++-XX-dev libc++abi-XX-dev
```

注意中间的`XX`是版本号，例如如果安装的是clang-18，那么使用的命令就应该是：

```cpp
sudo apt-get install libc++-18-dev libc++abi-18-dev
```

然后就可以尝试进行编译了，可以使用一个明显需要使用libc++的代码。例如，本博客写成的时候，g++只支持到c++20，缺少C++23的`std::print`。而clang++-18支持`std::print`，这里就可以使用下面的代码

```cpp
// a.cc
#include <print>

int main() {
  std::print("Hello {}\n", "world");
  return 0;
}
```

然后在编译的时候，需要使用`-std=libc++`的选项进行编译。

```bash
clang++ a.cc -std=c++23 -stdlib=libc++
```

## 配置clang默认使用libc++

如果希望clang持久性地使用libc++，可以选择下面的方式：

1. 设置别名：在`.bashrc`或者`.zshrc`里面，设置：

    ```bash
    alias clang++ = "clang++ -stdlib=libc++"
    ```
2. 使用脚本封装，在PATH里面，创建一个脚本文件，赋予执行权限后写入：

    ```bash
    #!/bin/bash
    clang++ -stdlib=libc++ "$@"
    ```
3. 通过环境变量，在`.bashrc`和`.zshrc`里面写入（但这种方式同样也会影响本地的g++编译器）。

    ```bash
    export CXXFLAGS="-stdlib=libc++"
    ```
4. 以上的方法实际上都或多或少存在一些问题，因此还有一个解决办法，可以让g++默认使用libstdc++，让clang++默认使用libc++。就是直接从源码安装clang，但是在安装clang的过程中，修改`CLANG_DEFAULT_CXX_STDLIB`字段值为`libc++`。位置在`llvm-project/clang/CMakeLists.txt`下。默认情况下，会跟随系统使用stdlib库，可以通过修改这一条直接指定编译出来的clang使用libc++。

