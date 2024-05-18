---
title: 在linux上远程分享命令行
description: 不使用那些付费的软件，仅仅使用tmux和ssh两个工具，就可以在两台机器上远程共享命令行。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Linux/2.jpg
category: 开发工具
published: 2023-10-20
tags: [Linux]
---
-----

如果要共享桌面，那么市面上有很多共享桌面的软件，使用远程会议即可。但是这些软件很多都需要付费。在linux下，实际上有更加简便的方式，利用最常用的ssh和tmux即可。

## Installation

需要安装tmux和ssh两个命令行软件。

----

### tmux

为了照顾新手，先来介绍一下tmux和ssh的下载安装。

首先需要安装tmux，tmux是一个非常流行的终端复用工具，除了本文推荐的用法，也强烈建议在日常工作中使用。

这里仅列举arch linux系列和debain（例如ubuntu，mint, deepin等）下的安装方式：

* 在arch linux系列下使用`sudo pacman -S tmux`​
* 在debain系列下使用`sudo apt install tmux`​

-------

### ssh

一般来说，现代的系统基本已经预装了ssh，可以远程登录他人的命令行。但是对于要共享命令行的人来说，还需要安装sshd，并且开启sshd。

下载安装sshd：

* 在arch linux系列下使用`sudo pacman -S openssh`​
* 在debain系列下使用`sudo apt-get install sshd`​

# Share Command Line

对于要共享命令行的人，依照以下步骤：

1. 使用`ifconfig`​等命令，告诉要分享的人如何使用ssh连接到你的服务器。
2. 使用`tmux`​，开启tmux终端。确认tmux终端的会话名。

> 如果你不知道tmux的会话名如何设置，那么就仅打开1个tmux的会话。

对于要接受共享命令行的人：

1. 首先使用ssh连接到对方的服务器。
2. 然后使用`tmux list-session`​命令，查看对方的tmux会话。

    > 例如，你可能会看到这样的提示：
    >
    > ​`0: 1 windows (created Sun Oct 8 10:56:14 2022) (attached)`​
    >
    > 那么tmux的会话名就是`0`​。
    >
    > 或者有时对方可能会设置会话名，那么你可能看见
    >
    > ​`ForShare: 1 windows (created Sun Oct  8 11:02:31 2023) (attached)`​
    >
    > 那么tmux的会话名就是`ForShare`​。
    >
3. 然后按照对方告知的tmux会话名，使用`tmux attach -t <session-name>`​即可看到对方的终端。对方所做的操作，也会展示到你那里。

> 注意：因为使用ssh的原因，你可以对被共享的服务器进行操作。在退出时，不要使用`exit`​命令。你需要让对方告知退出`tmux`​的方法，然后再在终端中使用ssh退出终端。
