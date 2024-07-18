---
title: lambda演算简单入门
description: lambda演算是20世纪30年代由Alonzo Church提出的一种计算模型，这种计算模型在计算理论，类型论和函数式编程中都得到了广泛应用。是思考计算问题的一个很好用的工具。本篇博客来简单介绍一下这个模型。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Math/9.jpg
category: 技术原理
published: 2024-07-18
tags: [Math]
---
----------------
## 基本规则

在lambda演算中，只有三项事物：

1. 变量，Variable，这里它可以是抽象的任何事物，使用$x, y, z$来表示。
2. 抽象，Abstract，它是一个完整的函数定义，用$\lambda$开头作为变量，后面跟着表达式。
3. 应用，Application，它指的是，把函数抽象应用在参数上。

例如，类似于$\lambda x y \rightarrow x y$这样的表达式就是一个lambda，里面的`x`和`y`就是变量，但是要注意的是，右边的`xy`不是x跟y相乘的意思，而是把`x`的抽象应用到`y`上。

假设我们已经有一个抽象，用`succ`来标记，然后有一个自然数0，我们可以把上面的$\lambda x y \rightarrow x y$，给应用到`succ`和自然数0上：

$$
(\lambda x y \rightarrow x y)\  \rm{succ} \ 0
$$

这里面，我们把lambda右边的x替换为`succ`，y替换为0，就得到$\rm{succ}\ 0$，就是1。

## 修改一下数学表达方式

你可能会很奇怪，这到底有什么用？为了要描述清楚这个问题，我需要前面的数学表达方式稍微调整一下，否则一会会非常混乱。

我们给每一个参数前面加一个$\lambda$符号，每个参数用`.`分隔，然后用一个箭头$\rightarrow$，后面跟着抽象表达。

对于前面的例子，写成这样：

$$
\lambda x.\lambda y \rightarrow x(y)
$$

需要额外一提的是，如果一个函数的参数和抽象的形式是一样的，那么着两个函数就是一样的，而不需要考虑字符是否相同，例如下面的两个函数是一样的函数。

$$
\lambda x.\lambda y \rightarrow x \ y\\
\lambda a.\lambda b \rightarrow a \ b

$$

然后，忘掉前面自然数和succ的这个例子，我们用纯粹的lambda演算来定义一些东西。

## 丘奇数

我们从很小的时候就学过算数，从小学一年级的第一堂课起，我们就知道了自然数这个东西，有没有思考过这样一件事，就是到底什么是数，0是什么，1又是什么？

对于这个问题的回答，每一种数学思考会得到不同的答案。在lambda演算这里，数可以代表这样一种东西，就是它本身就一个函数。把苹果带入3这个函数得到的是3个苹果，把x带入到2则得到2个x，至于这个x本身是什么，倒是无关紧要的东西。

把数看成是某种函数，这个就是丘奇数的想法，更具体一点数，丘奇规定了下面的丘奇数：

$$
\begin{aligned}
\bar{0} &= \lambda s.\lambda x \rightarrow x \\
\bar{1} &= \lambda s.\lambda x \rightarrow s \ x \\
\bar{2} &= \lambda s.\lambda x \rightarrow s \ (s \ x) \\
& ... \\
\bar{n} &= \lambda s.\lambda x \rightarrow s \ (s^{n-1} \ x) \\
\end{aligned}
$$

需要注意的是，这其中的s和x都是某种抽象，但是察觉s实际上是某种自增运算。由于它是一种抽象，所以这里定义出来的数都需要加上一个横杠。

然后，我们其实可以定义这种丘奇数的自增运算。它是下面的形式：

$$
\rm{SUCC} := \lambda n.\lambda f.\lambda x \rightarrow f ( n \ f \ x)
$$

我们尝试着进行`SUCC 1`，来看看会发生什么：

$$
\rm{succ} \ \bar{1} = (\lambda n.\lambda f.\lambda x \rightarrow f ( n \ f \ x)) (\lambda s.\lambda x \rightarrow s \ x)
$$

这里我们把前面的$\lambda n$消去（因为带入了1个参数），把$\bar{1}$带进到后面的n里面去，注意在后面的`f ( n f x)`里面，把n换掉之后，就变成了

$$
\rm f (n \ f \ x) = f ((\lambda s.\lambda x \rightarrow s \ x) \ f \ x) = f\ ( f\ x)
$$

所以$\rm succ\ \bar{1}$就变成了:

$$
\rm succ\ \bar{1} = \lambda f.\lambda x \rightarrow f\ (f\ x)
$$

我们可以发现，正好就是$\bar{2}$的形式，因此，证明了$\rm succ \ \bar{1} = \bar{2}$。

类似地同理可以证明$\rm succ \ \bar{n} = \overline{n+1}$，具体的证明过程这里不详细写了。

丘奇还给了两个丘奇数相加的函数，是下面的形式：

$$
add := λm.λn.λf.λx.\rightarrow m \ f\ (n\ f\ x)
$$

具体的证明过程这里不详细证了。

## 丘奇数的意义

现实生活中的自然数当然不可能使用丘奇数，但是对于数学分析或者实分析比较了解的人，看到前面的过程应该能够很快理解丘奇数厉害在哪里，至少应该能否很快反应过来，就是丘奇数与自然数是同构的，这里的同构指的是范畴上的同构，这个证明过程虽然可能比较复杂，但是道理却很容易想明白。

一旦确定了丘奇数和自然数的同构关系，那么任何自然数域中得到的结论，在丘奇数中也会有一个镜像的结论。但不要忘记，任何一个丘奇数都是二元函数这个事情。

## 布尔值与IF

有丘奇数这个例子在前面，我们可以定义出更多的东西出来，例如bool值：

$$
\begin{aligned}
\rm True&:= \lambda x.\lambda y \rightarrow x \\
\rm False &:= \lambda x.\lambda y \rightarrow y
\end{aligned}
$$

有了IF之后，也可以定义出IF条件判断：

$$
\rm IF:=\lambda b.\lambda x.\lambda y \rightarrow b \ x \ y
$$

来尝试带入$\rm IF\ True\ \bar{2}\ \bar{5}$试一下：

$$
\begin{aligned}
\rm IF\ True\ \bar{2}\ \bar{5} 
&= \rm (\lambda b.\lambda x.\lambda y \rightarrow b \ x \ y) \ True \ \bar{2} \ \bar{5} \\
&= \rm True \ \bar{2} \ \bar{5} \\
&= \rm (\lambda x.\lambda y \rightarrow x)\ \bar{2} \ \bar{5} \\
&= \bar{2} 
\end{aligned}
$$

## For循环

依靠lambda演算，也可以定义出for循环，把for循环理解为三个参数的函数，第一个参数是n，需要代入丘奇数，作为循环的次数，第二个参数是f，代表重复应用的函数，第三个参数是x，代表对谁进行重复应用，for循环可以定义成下面这样：

$$
For:= \lambda n.\lambda f.\lambda x \rightarrow n (\lambda g.\lambda y\rightarrow g\ (f\ y))\ Id \ x
$$

其中的$\rm {Id}$表示的是恒等映射，形式为：

$$
Id:=\lambda x\rightarrow x
$$

这里关键是我们要弄懂中间的$n (\lambda g.\lambda y\rightarrow g\ (f\ y))\ Id$，仍然要注意这里的n需要代入丘奇数，我们可以先代入一个$\bar{0}$试一下，$\bar{0}$的定义是$\lambda s.\lambda x \rightarrow x$，那么这样一来，$(\lambda s.\lambda x \rightarrow x) (\lambda g.\lambda y\rightarrow g\ (f\ y))\ Id$得到的就是$Id$，也就是循环了0次就等于没有循环。

代入$\bar{1}$试一下，$\bar{1} = \lambda s.\lambda x \rightarrow s \ x$，那么$(\lambda s.\lambda x \rightarrow s \ x) (\lambda g.\lambda y\rightarrow g\ (f\ y))\ Id$，得到的是$(\lambda g.\lambda y\rightarrow g\ (f\ y))\ Id$，这里面把g代入$Id$，得到就是$f \ y$，这样一来的话：

$$
(\lambda s.\lambda x \rightarrow s \ x) (\lambda g.\lambda y\rightarrow g\ (f\ y))\ Id = (\lambda g.\lambda y\rightarrow g\ (f\ y))\ Id = \lambda y \rightarrow f\ y
$$

注意这里面其实出现了一个闭包的问题，这里我们暂时不讨论这个，只需要知道最终出现的是$\lambda y \rightarrow f\ y$，把x代入自然得到的就是$f \ x$。

也就是说当$\bar{1}$代入n的时候，得到的确实是把f应用到x一次的结果。

那如果代入的是$\bar{2}$呢，这个时候就不是把$(\lambda g.\lambda y\rightarrow g\ (f\ y)) $应用到$Id$了，而是应用到刚才得到的$\lambda y \rightarrow f\ y$，可以得到$\lambda y \rightarrow f \ (f\ y)$，也就是把参数应用两次f函数之后的结果。

依次类推，可以知道代入$\bar{n}$得到的就是循环n次得到的结果。

