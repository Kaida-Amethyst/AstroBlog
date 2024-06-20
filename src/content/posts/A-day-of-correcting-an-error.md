---
title: 改错记
description: 在公司改bug时的一见趣事
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/AI/cat-1.png
category: 个人成长
published: 2022-01-24
tags: [Life]
---

----------------

今天在公司修bug，偶然间发现了一份神奇的代码，看日期是有些年头了。这份代码的神奇之处在于，短短二十几行代码里做到了平均每行都有一到两个错误😅

但是它居然能运行！还不出错！我当时就想出了这张图：

<img src="https://b3logfile.com/siyuan/1644568593533/assets/image-20240619220907-7204h4h.png" width=50% height=50%>

不过最近公司换了新的芯片架构，这份神奇的代码就跑不动了。修改这份神奇的代码的工作就落在了我的肩上。

我左思右想，也不知道该怎么修改这份神奇的代码，于是就找了领导来看。领导看了也是抓耳挠腮，看了半天不知道怎么下手，只觉得好像在哪里见过。一筹莫展之际就说“这样吧，用`git blame`可以查到负责人的，看看这弱智代码是哪个沙雕写得”。

好嘞，于是我飞快地在键盘上打入了`git blame *`。结果蹦出来的竟然是……

领导的名字……(⊙…⊙)


<img src="https://b3logfile.com/siyuan/1644568593533/assets/image-20240619220923-q7nui8p.png" width=50% height=50%>
