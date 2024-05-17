---
title: 一些有用的linux命令行工具
description: 介绍了bat lsd rg btop fzf等一些有用的linux命令行工具
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Linux/4.jpg
category: Tools
published: 2023-04-02
tags: [Linux]
---

## bat

github位置：[https://github.com/sharkdp/bat](https://github.com/sharkdp/bat)

当我们希望查看一个文件的内容时，我们有时会实用`cat`这个工具，但是`cat`基本只能把文本的内容不加修饰地打印出来，当我们查看的是代码或者像json这样的有结构的文件时，因为缺少格式和高亮，可能会显得有些“素”。`bat`则可以用来替代`cat`，因为它提供了一些格式和高亮之类的功能，可以看下面的示例来比较一下：


![image](https://b3logfile.com/siyuan/1644568593533/assets/image-20240512221356-h0ywujz.png)

### 安装方法

一般来说，可以使用apt或者pacman这样的工具来进行安装：

```shell
# Ubuntu
sudo apt install bat

# Arch
pacman -S bat
```

如果机器上有rust，也可以利用cargo安装：

```shell
cargo install --locked bat
```

需要稍微注意一下的是，`bat`这个工具的位置，如果不在`/usr/bin`目录下的话，可能在`~/.local/bin`下，如果是通过cargo安装的，可能在`~/.cargo/bin`下。

---

## lsd

github位置：[https://github.com/lsd-rs/lsd](https://github.com/lsd-rs/lsd)

类似于默认的`ls`，不过`lsd`为目录的查看提供了更加丰富的表现力，例如添加了图表或者颜色：

![image](https://b3logfile.com/siyuan/1644568593533/assets/image-20240512222105-xh5ykt3.png)

### 安装方法

与bat一样，可以通过包管理器安装：

```shell
# Ubuntu
sudo apt install lsd

# Arch
pacman -S lsd
```

或者通过cargo安装，因为lsd也是一个rust项目：

```shell
cargo install lsd
```

---

## ripgrep (rg)

github位置：[https://github.com/BurntSushi/ripgrep](https://github.com/BurntSushi/ripgrep)

ripgrep类似于grep，但是性能更高且更加易用。

看一下截图：

![image](https://b3logfile.com/siyuan/1644568593533/assets/image-20240512224810-uj0hoog.png)


### 安装方法

使用包管理安装：

```shell
# Ubuntu
sudo apt-get install ripgrep

# Arch
sudo pacman -S ripgrep
```

ripgrep也是rust项目，因此也可以通过cargo安装

```shell
cargo install ripgrep
```

---

## btop

github位置：[https://github.com/aristocratos/btop](https://github.com/aristocratos/btop)

btop类似top命令，但是界面要更加好看，且内置了许多主题：

看一下效果：

![image](https://b3logfile.com/siyuan/1644568593533/assets/image-20240512225147-xmmlclh.png)


### 安装方法

btop是一个c++项目，安装起来稍微麻烦一点，比较建议的方法是按照github的介绍通过源码安装：

首先去[btop latest release](https://github.com/aristocratos/btop/releases/latest)下载最新的源码，在源码的目录里运行下面的命令：

```shell
sudo make install
```

---

## fzf

github位置：[https://github.com/junegunn/fzf](https://github.com/junegunn/fzf)

fzf是一个快速的文件查找器，当你知道一个文件的名称或者大体名称，但是一时间找不到具体位置时，就可以使用fzf了。


![image](https://b3logfile.com/siyuan/1644568593533/assets/image-20240512225746-2emobtw.png)



### 安装方法

可以通过包管理器直接安装：

```shell
# Ubuntu
sudo apt-get install fzf

# Arch
sudo pacman -S fzf
```

---

## Zellij

github位置：https://github.com/zellij-org/zellij

zellij是一个终端复用工具，它跟tmux一样，但是相较于tmux而言，zellij的界面要更加cool，而且学习曲线要平缓地多。

看一下zellij的显示效果：


![image](https://b3logfile.com/siyuan/1644568593533/assets/image-20240512230248-6ltny7r.png)


### 安装方法

zellij基本不需要依赖库，所以可以通过github页面上直接下载可执行文件放到机器上即可。

或者通过cargo安装：

```shell
cargo install --locked zellij
```
