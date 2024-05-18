---
title: Ubuntu 20.04安装最新版gcc
published: 2022-03-16
description: 在ubuntu 20.04上通过源码安装最新版本的gcc
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Linux/1.jpg
category: 开发工具
tags: [Linux, Ubuntu]
---

在新安装的ubuntu系统上，通过源码，安装最新版的gcc。

> 需要说明的是，本篇笔记写成时，gcc的最新版本是gcc-12，因此后面在新建目录以及链接的时候，都使用了`gcc12`或`gcc-12`。如果再次安装的时候是更新的版本，最好使用新的名称。

## Prerequirement

先安装必要的部件，根据需要，如果有已经安装完毕的，可以略过。

```shell
$ sudo apt install -y build-essential git libmpc-dev flex
```

## 下载源码

```shell
$ git clone git://gcc.gnu.org/git/gcc.git gcc && pushd gcc/
```

## 配置编译环境

下载之后，进入到gcc目录下，进行配置，这里我们指定把gcc给安装到`/usr/local/gcc12`里去，这里我们的系统是64位的，因此直接用`--disable-multilib`，只安装64位。

```shell
$ ./configure --prefix=/usr/local/gcc12 --with-system-zlib --disable-multilib
$ make -j6 # 编译，略耗时
$ sudo make installls 
```

之后，将 `/usr/local/gcc12/bin` 下的可执行程序全都软链接到 `/usr/bin/`

```shell
$ sudo ln -sf /usr/local/gcc12/bin/gcc /usr/bin/gcc-12
$ sudo ln -sf /usr/local/gcc12/bin/g++ /usr/bin/g++-12
$ sudo ln -sf /usr/local/gcc12/bin/gcc-ar /usr/bin/gcc-ar-12
$ sudo ln -sf /usr/local/gcc12/bin/gcc-nm /usr/bin/gcc-nm-12
$ sudo ln -sf /usr/local/gcc12/bin/gcc-ranlib /usr/bin/gcc-ranlib-12
$ sudo ln -sf /usr/local/gcc12/bin/gcov /usr/bin/gcov-12
$ sudo ln -sf /usr/local/gcc12/bin/gcov-dump /usr/bin/gcov-dump-12
$ sudo ln -sf /usr/local/gcc12/bin/gcov-tool /usr/bin/gcov-tool-12
```

## 配置多版本gcc共存

```shell
$ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 120 --slave /usr/bin/gcov gcov /usr/bin/gcov-12
$ sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-12 120
$ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-9 90 --slave /usr/bin/gcov gcov /usr/bin/gcov-9
$ sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-9 90
```

之后，如果要更改默认版本，使用如下的命令

```shell
$ sudo update-alternatives --config gcc

There are 2 choices for the alternative gcc (providing /usr/bin/gcc).

  Selection    Path             Priority   Status
------------------------------------------------------------
* 0            /usr/bin/gcc-12   120       auto mode
  1            /usr/bin/gcc-12   120       manual mode
  2            /usr/bin/gcc-9    90        manual mode

Press <enter> to keep the current choice[*], or type selection number: 
```

上面是修改`gcc`的默认版本，同理也可以修改`g++`的默认版本。

## 参考链接

[1]: https://blog.byhi.org/archives/586372ad.html

[2]: https://gcc.gnu.org/

[3]: https://gcc.gnu.org/git.html
