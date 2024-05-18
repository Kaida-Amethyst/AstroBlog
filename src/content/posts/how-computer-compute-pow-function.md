---
title: 计算机是如何计算pow函数的
description: 从数学的角度，解释计算机是如何计算pow函数的
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Math/6.jpg
category: 技术原理
published: 2022-04-29
tags: [Math]
---

定义函数：

```c
float powf(float x, float y);
```

要求计算：

$$
z = x ^y \tag{1}
$$

注意这里的两个数都是浮点数，因此不可能通过循环相乘的办法得到，可以使用其它办法，把上面的公式两边取对数：

$$
\log_2z = \log_2x^y = y\times\log_2x \tag{2}
$$

因此只要先求出$\log_2x$然后再乘以$y$，然后将得到的值再作为2的指数进行运算即可。也就是说：

$$
z = 2^{y\times \log_2x}
$$

这里有两个问题：

1. 如何去求这个$\log_2x$？
2. $y\times \log_2x$有可能不是一个整数，那么将其作为2的指数又该怎么去计算？

## 求解$\log_2x$：

那么第一个的问题就在于怎么去计算这个$\log_2x$，我们可以使用泰勒公式，将其展开，但是注意，这并不是一个经典的泰勒公式的问题，我们需要进行推导。

直接求$\log_2x$并不好求，我们可以利用$\ln x$来求。

我们有如下的泰勒公式：

$$
\begin{aligned}
\ln (1+x) &= x-\frac{1}{2}x^2+\frac{1}{3}x^3-\frac{1}{4}x^4+\dots \\
&= \sum_{k=1}^{n}\frac{(-1)^k}{k}x^k
\end{aligned} \tag{3}
$$

上述公式的理论范围为$x\in \mathbb{R}, x\in [-1, +\infin)$，但是实际上考虑到上述公式的推导过程，这个公式只有在$x$在0附近才会有一个比较好的精度，因此通常限定$x$的范围为$x\in[-1, 1]$，另外注意，这里是$\ln(1+x)$，我们想要求的是$\ln x$，所以通常的做法是：

$$
\begin{aligned}
&\text{let }t = x-1, \text{then } x=t+1, \text{therefore}\\
&\ln x = \ln(t+1) = \sum_{k=1}^n\frac{(-1)^k}{k}t^k
\end{aligned}  \tag{4}
$$

此时$x$在$x\in (0, 2)$时有比较好的精度。但是如果$x$过大的话，就会有非常大的精度下降的问题。

除此之外，我们还有另外一个公式：

$$
\begin{aligned}
\ln x &= 2\big[ (\frac{x-1}{x+1})+\frac{1}{3}(\frac{x-1}{x+1})^3+\frac{1}{5}(\frac{x-1}{x+1})^5+\dots\big]\\
&= 2\sum_{k=0}^{n}\frac{1}{2k+1}(\frac{x-1}{x+1})^{2k+1}
\end{aligned}  \tag{5}
$$

这个公式的推导方法如下：

$$
\ln(1+x) = \sum_{k=1}^n\frac{(-1)^k}{k}x^k  \text{    when }x\in[-1,+\infin) \tag{6}
$$

将其中的$x$代换成$\frac{1}{x}$可以得到：

$$
\ln(1+\frac{1}{x}) =\ln(\frac{x+1}{x})= \sum_{k=1}^n\frac{(-1)^k}{k}\frac{1}{x^k}\tag{7}
$$

而$x$的区间范围为：

$$
\frac{1}{x}\in(-1,+\infin)\Longrightarrow x\in(-\infin,-1)\cup(0,+\infin) \tag{8}
$$

我们将$(6)$中的$x$再度替换：

$$
\ln(1-\frac{1}{x}) = -\sum_{k=1}^n\frac{1}{kx^k}\tag{9}
$$

然后对$(9)$的两边做处理：

$$
\begin{aligned}
&\ln(1-\frac{1}{x}) = -\sum_{k=1}^n\frac{1}{kx^k}\\
\Rightarrow &-\ln(1-\frac{1}{x})=-\ln(\frac{x-1}{x})=\ln{\frac{x}{x-1}} = \sum_{k=1}^n\frac{1}{kx^k}
\end{aligned}  \tag{10}
$$

照例分析一下$x$的区间范围，不难得出：

$$
-\frac{1}{x}\in(-1,+\infin) \Longrightarrow x\in(-\infin,0)\cup (1,\infin)\tag{11}
$$

然后把$(7)$式和$(10)$式相加，得到：

$$
\begin{aligned}
&\ln(\frac{x+1}{x}) + \ln(\frac{x}{x-1})=\ln(\frac{x+1}{x-1}) \\
&\sum_{k=1}^n\frac{(-1)^k}{kx^k} + \sum_{k=1}^n\frac{1}{kx^k} = 2\sum_{k=0}^n{\frac{1}{2k+1}\frac{1}{x^{2k+1}}}
\end{aligned}
$$

因而：

$$
\ln(\frac{x+1}{x-1}) = 2\sum_{k=1}^n{\frac{1}{2k+1}\frac{1}{x^{2k+1}}} \tag{12}
$$

再将$(9)$和$(11)$对于$x$的分析进行一个合并，不难得出：

$$
x\in(-\infin,-1)\cup(1,\infin)\text{ or } |x|>1 \tag{13}
$$

然后，我们做如下的处理：

$$
\text{ let } \frac{x+1}{x-1} = z
$$

这里分析一下这个$z$的值域。当$x> 1$时，我们有：

$$
\begin{aligned}
\lim_{x\rightarrow1}\frac{x+1}{x-1} &= +\infin \\
\lim_{x\rightarrow +\infin}\frac{x+1}{x-1} &= 1
\end{aligned}
$$

上面的两式告诉我们两点：

$$
\begin{aligned}
&\text{1. }\frac{x+1}{x-1} \text{ 在}x\in(1,+\infin)\text{上连续。}\\
&\text{2. }\frac{x+1}{x-1} \in(1, +\infin) \text{ 当} x\in(1,+\infin)\text{时。}
\end{aligned}  \tag{14}
$$

再来看$x\leq -1$的情形：

$$
\begin{aligned}
&\text{1. } \frac{x+1}{x-1}\big|_{x=-1} = 0 \\
&\text{2. } \lim_{x\rightarrow-\infin}\frac{x+1}{x-1} = 1
\end{aligned}
$$


因而：

$$
\begin{aligned}
&\text{1. }\frac{x+1}{x-1} \text{ 在}x\in(-\infin,-1)\text{上连续。}\\
&\text{2. }\frac{x+1}{x-1} \in(0,1) \text{ 当} x\in(-\infin,-1)\text{时。}
\end{aligned}  \tag{15}
$$

结合$(14)$和$(15)$的结论，我们得出：

$$
\frac{x+1}{x-1} = z \in(0,+\infin) \tag{16}
$$

于是我们得出我们最终的结论：

$$
\begin{aligned}
\ln x &= 2\big[ (\frac{x-1}{x+1})+\frac{1}{3}(\frac{x-1}{x+1})^3+\frac{1}{5}(\frac{x-1}{x+1})^5+\dots\big]\\
&= 2\sum_{k=0}^{n}\frac{1}{2k+1}(\frac{x-1}{x+1})^{2k+1} \\
& (x\in \mathbb{R} \text{ and } x > 0)
\end{aligned}  \tag{17}
$$

为什么$(17)$式比$(4)$式要更好：

理论上来说，只要展开到无穷项，就可以获得相同的精度，但问题在于我们通常不可能真的将其展开到无穷项，而只是展开到有限项求一个近似。$(4)$式的问题在于，假定我们只展开到第$k-1$项，那么它的余项为：

$$
o(\frac{(-1)^k}{k}t^k) \text{ where } t = x - 1
$$

这个余项，实际上就是误差项，它的问题在于，如果$x\geq 2$，那么$t>1$，这会使得这最后一项产生非常严重的发散问题，从而造成极大的误差，因此公式$(3)$只有在$x\in(0,2)$时才会有比较好的精度。

再来看式$(17)$我们发现：

$$
x\geq 0,\frac{x-1}{x+1}\big|_{x=0} = -1, \lim_{x\rightarrow+\infin}\frac{x-1}{x+1} = 1，\frac{x-1}{x+1}\in[-1,1]
$$

假定展开到第$k$项，那么它的余项是：

$$
o(\frac{1}{2k+1}(\frac{x-1}{x+1})^{2k+1})
$$

这一项可以保证始终在一个小区间内波动，此外它的单调性与$\ln x$也趋于一致，因此式$(17)$有更好的收敛性，从而在接收较大一点的数时仍然可以保持比较高的精度。不过因为通常展开的项数不够多，当接收的$x$过大时，其精度也会有相当程度的降低。

但是，既然已经知道这个公式在$x$越靠近0的时候有较高的精度，那么我们可以这么做：

1. 把$x$放缩到$[1,2]$区间内。假定放缩系数为$1/2^n$。
2. 利用公式计算$\ln (x/2^n)$，此时可以获得比较高的精度。
3. 考虑到$\ln(x/2^n) = \ln(x) - n\ln(2)$，因此不难得出：

    $$
    \ln(x) = \ln \frac{x}{2^n} + n\ln(2) \tag{18}
    $$

4. 而这里的放缩系数实际上很好求，考虑到IEEE754下的浮点数标准，这里的$n$实际就可以是浮点数$x$的阶码，而$x/2^n$，实际上相当于把x的阶码替换为127（如果x是`float`），或者1023（如果x是`double`）。

另外，有关对数函数的计算，笔记中有价值的文章是：((20220708110318-f0uqjrt '计算机是怎样计算log的'))。

## 最终计算$x^y$

现在我们已经可以计算$\ln x$，那么要求$x^y$就很简单了：

$$
\begin{aligned}
&z = x^y \\
\Longrightarrow &\log_2 z = \log_2 x^y \\
\Longrightarrow &\log_2 z = y\log_2x\\
\Longrightarrow &\log_2 z = y\frac{\ln x}{\ln 2} \\
\Longrightarrow &z = 2^{y{\ln x}/{\ln 2}}
\end{aligned}
$$

## 算法思路

考虑到计算机实现的效率问题，需要把整个$\ln x$的计算做一个拆分：

$$
\text{ let }z = \frac{x-1}{x+1}，s=z^2
$$

假定我们将$\ln x$展开到第15项，那么：

$$
\begin{aligned}
\ln x &\approx  2z+\frac{2}{3}z^3+\frac{2}{5}z^5+\frac{2}{7}z^7+\frac{2}{9}z^9+\frac{2}{11}z^{11}+\frac{2}{13}z^{13}+\frac{2}{15}z^{15} \\
&= z(2+\frac{2}{3}z^2+\frac{2}{5}z^4+\frac{2}{7}z^6+\frac{2}{9}z^8+\frac{2}{11}z^{10}+\frac{2}{13}z^{12}+\frac{2}{15}z^{14}) \\
&=z(2+\frac{2}{3}s+\frac{2}{5}s^2+\frac{2}{7}s^3+\frac{2}{9}s^4+\frac{2}{11}s^5+\frac{2}{13}s^6+\frac{2}{15}s^7)\\
&=z\times r
\end{aligned}
$$

其中：

$$
\begin{aligned}
r &= 2+\frac{2}{3}s+\frac{2}{5}s^2+\frac{2}{7}s^3+\frac{2}{9}s^4+\frac{2}{11}s^5+\frac{2}{13}s^6+\frac{2}{15}s^7\\
&=2+s(\frac{2}{3}+s(\frac{2}{5}+s(\frac{2}{7}+s(\frac{2}{9}+s(\frac{2}{11}+s(\frac{2}{13}+\frac{2}{15}s))))))
\end{aligned}
$$

考虑到我们最终要求的是$\log_2x$因此我们这里直接将$1/\ln2$乘到$r$上，这样一来就得到了系数：

$$
\begin{aligned}
L_1 &= \frac{2}{\ln 2} =  2.8853900817779268 \\
L_2 &= \frac{2}{3\ln 2}= 0.9617966939259757 \\
L_3 &= \frac{2}{5\ln 2}= 0.5770780163555853\\
L_4 &= \frac{2}{7\ln 2}= 0.41219858311113244\\
L_5 &= \frac{2}{9\ln 2}= 0.3205988979753252\\
L_6 &= \frac{2}{11\ln 2}=0.2623081892525388\\
L_7 &= \frac{2}{13\ln 2}=0.2219530832136867 \\
L_8 &= \frac{2}{15\ln 2}=0.19235933878519512
\end{aligned}
$$

因此把上面全部代入，就是：

$$
\begin{aligned}
\log_2x &= z\times r' \\
r'&=  L_1+s(L_2+s(L_3+s(L_4+s(L_5+s(L_6+s(L_7+L_8s))))))
\end{aligned}
$$

但是如果只是这样的话，当$x$非常大的时候，精度会非常不够，因此需要做一个放缩，也就是不要直接使用上面的公式，而是将$x$假定$x$很大的时候，缩小到1附近。

假定缩小$2^n$后在1附近，那么：

$$
\begin{aligned}
x' &= \frac{x}{2^n} \\
\log_2x' &= z\times r' \\
\log_2x &= \log_22^nx' = n+\log_2x'
\end{aligned}
$$

然后再将$\log_2x$与$y$相乘，再将其作为2的指数即可计算出最终结果：

$$
\begin{aligned}
\mu &= y\times \log_2x\\
z &= 2^\mu
\end{aligned}
$$

不过通常来说，这里的$\mu$很可能不是一个整数，那么我们怎么得到它的值？

## 求$2^\mu$的方法

同样地，恐怕需要利用泰勒公式，我们有如下的泰勒公式：

$$
\begin{aligned}
e^\mu &= 1+\mu+\frac{1}{2!}\mu^2+\frac{1}{3!}\mu^3+o(\frac{1}{4!}\mu^4) \\
&= \sum_{k=0}^n\frac{1}{k!}\mu^k
\end{aligned}  \tag{19}
$$

又因为$2 = e^{\ln 2}$，因此我们有：

$$
\begin{aligned}
\text{let } t = \mu\ln 2 \\
2^x = \sum_{k=0}^n\frac{1}{k!}t^k
\end{aligned}  \tag{20}
$$

但是这里我们再次遇到一个问题，$2^x$的泰勒展式的余项是一个指数形式，是发散的，要获得比较好的精度，上面的需要尽量靠近0才行，但是一般而言，这里的$t$可能会离0比较远，为了解决这个问题，就需要把得到的$t$缩小到0附近。但是不要用除法，而是使用减法：

$$
\begin{aligned}
\text{ let } \mu' &= \mu - \Delta\\
\text{then } \mu &= \mu' + \Delta\\
\text{hence } 2^\mu &= 2^{\mu'+\Delta} = 2^{\mu'} \times 2^\Delta
\end{aligned}
$$

上面的$\Delta$需要是一个整数，这样在最后计算$2^\mu$的时候，就可以使用scalbnf了。而对于$2^{\mu'}$，只需要利用式$(20)$即可。

这样，最终的$x^y$的值就求出来了。
