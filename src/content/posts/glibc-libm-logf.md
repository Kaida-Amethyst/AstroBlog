---
title: glibc源码剖析 - logf
description: 剖析一下glibc中logf的实现，这个代码的解读需要不少数学知识
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Math/2.jpg
category: Principle
published: 2023-04-02
tags: [C, glibc]
---
----

## 前言

log函数是一个比较复杂的函数，它在计算机上的实现方式，非常值得计算科学的人来仔细研究。

-----

## 思路

对于任意底的对数$\log_k$，由于我们有换底公式：

$$
\log_k{x} = \frac{\ln{x}}{\ln k} \tag{1}
$$

而上面的$\ln k$是一个定值，因此对于任意底的对数，我们只需要计算$\ln x$就可以了。

### $\ln x$的计算

高等数学中，我们学过如下的泰勒展式：

$$
\begin{aligned}
\ln (1+x) &= x-\frac{1}{2}x^2+\frac{1}{3}x^3-\frac{1}{4}x^4+\dots \\
&= \sum_{k=1}^{\infty}\frac{(-1)^k}{k}x^k
\end{aligned} \tag{2}
$$

那么假设我们想要求$\ln x$，自然只需要令$t=x-1$，那么：

$$
\begin{aligned}
\ln(x) = \ln (1+t) = \sum_{k=1}^{\infty}\frac{(-1)^k}{k}t^k
\end{aligned} \tag{2}
$$

但是上面的公式其实不太好，因为在计算中，我们通常不可能把上式展开真的展开到无穷项，而只是将其展开有限项，假定将其展开到第$n$项，那么上式带Peano余项的泰勒展式可以写作：

$$
\begin{aligned}
\ln (1+t) &= t-\frac{1}{2}t^2+\frac{1}{3}t^3-\frac{1}{4}t^4+\dots \\
&= \sum_{k=1}^{n}\frac{(-1)^k}{k}t^k + o(t^n)
\end{aligned} \tag{3}
$$

注意这里的Peano余项是$o(t^n)$，就是误差项，也就是说如果$t>1$，也就是$x > 2$，上面公式的误差可能会随着$t$或者$x$的增大而变得非常大。

不过我们还有下面的公式：

$$
\begin{aligned}
\ln x &= 2\big[ (\frac{x-1}{x+1})+\frac{1}{3}(\frac{x-1}{x+1})^3+\frac{1}{5}(\frac{x-1}{x+1})^5+\dots\big]\\
&= 2\sum_{k=0}^{\infty}\frac{1}{2k+1}(\frac{x-1}{x+1})^{2k+1}
\end{aligned}\tag{4}
$$

这里对上面的公式给一个不严谨的证明：

把(2)式中的$x$分别换成$\frac{1}{x}$和$-\frac{1}{x}$，可以得到：

$$
\begin{aligned}
\ln(1+\frac{1}{x}) &=\ln(\frac{x+1}{x})= \sum_{k=1}^\infty\frac{(-1)^k}{k}\frac{1}{x^k} \\ 
\ln(1-\frac{1}{x}) &= \ln(\frac{x-1}{x}) = -\sum_{k=1}^\infty\frac{1}{kx^k}
\end{aligned}
$$

注意：

$$
\begin{aligned}
\ln(\frac{x-1}{x}) &= -\sum_{k=1}^\infty\frac{1}{kx^k} \\
\Longrightarrow -\ln(\frac{x-1}{x}) &= \sum_{k=1}^\infty\frac{1}{kx^k} \\
\Longrightarrow \ln(\frac{x}{x-1}) &= \sum_{k=1}^\infty\frac{1}{kx^k} \\
\end{aligned}
$$

然后：

$$
\begin{aligned}
&\ln(\frac{x+1}{x}) + \ln(\frac{x}{x-1})=\ln(\frac{x+1}{x-1}) \\
&\sum_{k=1}^n\frac{(-1)^k}{kx^k} + \sum_{k=1}^n\frac{1}{kx^k} = 2\sum_{k=0}^n{\frac{1}{2k+1}\frac{1}{x^{2k+1}}}
\end{aligned}
$$

这样我们得到的是：

$$
\ln(\frac{x+1}{x-1}) = 2\sum_{k=0}^n{\frac{1}{2k+1}\frac{1}{x^{2k+1}}}
$$

令$z = \frac{x+1}{x-1}$，那么$x = \frac{z-1}{z+1}$，代入到上式，就得到(4)式了。

这个公式的好处在于，假设我们将其展开到n项，那么：

$$
\begin{aligned}
\ln x &= 2\big[ (\frac{x-1}{x+1})+\frac{1}{3}(\frac{x-1}{x+1})^3+\frac{1}{5}(\frac{x-1}{x+1})^5+\dots\big]\\
&= 2\sum_{k=0}^{n}\frac{1}{2k+1}(\frac{x-1}{x+1})^{2k+1} + o((\frac{x-1}{x+1})^{2k+1})
\end{aligned}\tag{5}
$$

它的余项是$o((\frac{x-1}{x+1})^{2k+1})$，我们有：

$$
\lim_{x\rightarrow+\infty}(\frac{x-1}{x+1})^{2k+1} = 1
$$

因此它的余项是可以保证误差始终在1的范围内的。

但是还是不够，因为1的误差还是太大，根据上面对与余项的分析，这个$x$的值实际上要越靠近1越好。越靠近1，精度就越高。

考虑下IEE754的浮点数标准，我们知道在计算机中，任意的浮点数，在计算机中都表示为下面的形式：

$$
x = \rm1.xxx \times 2^{exp}
$$

意思就是我们可以直接通过位运算来获取这个exp（或称阶码），和前面的`1.xxx`​（或称尾数）。

既然浮点数直接就是这个形式，那么我们直接使用前面的这个`1.xxx`​不就行了？

我们用前面的公式去计算$\ln{\rm 1.xxx}$，其实就相当计算$\ln \frac{x}{2^{exp}}$，那么根据对数公式：

$$
\ln(\frac{x}{2^{exp}}) = \ln x - \ln 2^{exp} = \ln x - exp\times \ln 2
$$

因此：

$$
\ln x = \ln(\frac{x}{2^{exp}}) + exp \times \ln 2
$$

那么这样$\ln x$就计算完成了，也就相当于任意底的对数，也就计算完成了。

## Glibc中的实现

```c
#define __HI(x) *(1+(int*)&x)
#define __LO(x) *(int*)&x

static double
ln2_hi  =  6.93147180369123816490e-01,	/* 3fe62e42 fee00000 */
ln2_lo  =  1.90821492927058770002e-10,	/* 3dea39ef 35793c76 */
two54   =  1.80143985094819840000e+16,  /* 43500000 00000000 */
Lg1  = 6.666666666666735130e-01,  /* 3FE55555 55555593 */
Lg2  = 3.999999999940941908e-01,  /* 3FD99999 9997FA04 */
Lg3  = 2.857142874366239149e-01,  /* 3FD24924 94229359 */
Lg4  = 2.222219843214978396e-01,  /* 3FCC71C5 1D8E78AF */
Lg5  = 1.818357216161805012e-01,  /* 3FC74664 96CB03DE */
Lg6  = 1.531383769920937332e-01,  /* 3FC39A09 D078C69F */
Lg7  = 1.479819860511658591e-01;  /* 3FC2F112 DF3E5244 */
zero =  0.0;

double __ieee754_log(double x) {
  double hfsq,f,s,z,R,w,t1,t2,dk;
  int k,hx,i,j;
  unsigned lx;

  hx = __HI(x);		/* high word of x */
  lx = __LO(x);		/* low  word of x */

  k=0;
  if (hx < 0x00100000) {			/* x < 2**-1022  */
    if (((hx&0x7fffffff)|lx)==0) 
        return -two54/zero;		/* log(+-0)=-inf */
    if (hx<0) return (x-x)/zero;	/* log(-#) = NaN */
    k -= 54; x *= two54; /* subnormal number, scale up x */
    hx = __HI(x);		/* high word of x */
  } 
  if (hx >= 0x7ff00000) return x+x;
  k += (hx>>20)-1023;
  hx &= 0x000fffff;
  i = (hx+0x95f64)&0x100000;
  __HI(x) = hx|(i^0x3ff00000);	/* normalize x or x/2 */
  k += (i>>20);
  f = x-1.0;
  if((0x000fffff&(2+hx))<3) {	/* |f| < 2**-20 */
    if(f==zero){
      if(k==0) 
        return zero;  
      else {
        dk=(double)k;
        return dk*ln2_hi+dk*ln2_lo;
      }
    }
    R = f*f*(0.5-0.33333333333333333*f);
    if(k==0) return f-R;
    else {
      dk=(double)k;
      return dk*ln2_hi-((R-dk*ln2_lo)-f);
    }
  }
  s = f/(2.0+f); 
  dk = (double)k;
  z = s*s;
  i = hx-0x6147a;
  w = z*z;
  j = 0x6b851-hx;
  t1= w*(Lg2+w*(Lg4+w*Lg6)); 
  t2= z*(Lg1+w*(Lg3+w*(Lg5+w*Lg7))); 
  i |= j;
  R = t2+t1;
  if(i>0) {
    hfsq=0.5*f*f;
    if(k==0) return f-(hfsq-s*(hfsq+R));
    else return dk*ln2_hi-((hfsq-(s*(hfsq+R)+dk*ln2_lo))-f);
  } else {
    if(k==0) return f-s*(f-R);
    else return dk*ln2_hi-((s*(f-R)-dk*ln2_lo)-f);
  }
}
```

## Source Code Analysis

> 注意：需要IEEE754 浮点数标准的知识。

先来分析在函数前面的一大堆常量，不难看出，它是我们在思路篇提到的一些常量。

```c
static double
// ln2的值，注意这里分成了两个部分，ln2_hi+ln2_lo才是ln2的实际值
ln2_hi  =  6.93147180369123816490e-01,	/* 3fe62e42 fee00000 */
ln2_lo  =  1.90821492927058770002e-10,	/* 3dea39ef 35793c76 */
// two54，就是2^54
two54   =  1.80143985094819840000e+16,  /* 43500000 00000000 */
// lg1实际上是2/3，但是注意它与实际的2/3有一点点差别
Lg1  = 6.666666666666735130e-01,  /* 3FE55555 55555593 */
// lg2实际上是2/5，也就是0.4，注意这里也是与标准的0.4有一点差别
Lg2  = 3.999999999940941908e-01,  /* 3FD99999 9997FA04 */
// lg3实际上是2/7
Lg3  = 2.857142874366239149e-01,  /* 3FD24924 94229359 */
// lg4实际上是2/9
Lg4  = 2.222219843214978396e-01,  /* 3FCC71C5 1D8E78AF */
// lg5实际上是2/11
Lg5  = 1.818357216161805012e-01,  /* 3FC74664 96CB03DE */
// lg6实际上是2/13
Lg6  = 1.531383769920937332e-01,  /* 3FC39A09 D078C69F */
// lg7实际上是2/15
Lg7  = 1.479819860511658591e-01;  /* 3FC2F112 DF3E5244 */
// 0，不赘述了
zero =  0.0;
```

然后我们来分析代码主体：

```c
double __ieee754_log(double x) {
  double hfsq,f,s,z,R,w,t1,t2,dk;
  int k,hx,i,j;
  unsigned lx;

  hx = __HI(x);		/* high word of x */
  lx = __LO(x);		/* low  word of x */
  /* 其它代码 */
}
```

我们首先定义了一堆变量，每个变量的具体含义，我们在后面用到的时候再解释。我们首先来解释`hx`​和`lx`​，它用到了glibc中的两个宏：

```c
#define __HI(x) *(1+(int*)&x)
#define __LO(x) *(int*)&x
```

我们知道在计算机中，一个`double`​是64位，一个`int`​是32位，这里的两个宏就是把一个`double`​的十六进制给拆成高低两个`int`​分别进行运算，因为我们无法对`double`​的十六进制位做直接处理。

接下来的代码，我们再来回顾IEEE754浮点数的知识，一个64位的浮点数，第一个高位是符号位，1代表负数，0代表正数。然后是11位代表阶码位，剩余的52位为尾数位。

另外，如果阶码全为0，或者全为1，那么这样的数被称为非规格数（subnormal）。全为1时，如果尾数全为0，那么它是无穷大，不为0，就是NaN（Not a number）。

如果阶码全为0的话，尾数也全为0，那么它就是0。

注意：**如果阶码全为0，尾数不为0的话，它是一个非规格化的小数。** 非规格化小数需要做特殊的处理。

```c
// 这里的k用来获取x的阶码  
k=0;
// hx < 0x00100000，它的意思其实就是x是一个非规格化数的意思，因为0x00100000
// 它的阶码就是1，这个数小，那么它的阶码肯定就是0了。
if (hx < 0x00100000) {			/* x < 2**-1022  */
  // 非规格化数也可能是0，如果x是0的话，那么应该直接return -inf.
  // 不过这里是用了一个负数（其实随便什么负数都可以）再除以一个0得到。
  if (((hx&0x7fffffff)|lx)==0) 
      return -two54/zero;		/* log(+-0)=-inf */
  // hx是负数，就是x是负数的意思，其实这里改写成x<0一点问题没有，x是负数
  // 那ln x就算不了，所以返回一个非数，这里是用0/0来表示的。
  if (hx<0) return (x-x)/zero;	/* log(-#) = NaN */
  // 如果不是上面的两种情况的话，那么x是一个非规格化的小数。
  // 不可以将其直接参与后面的运算，这里要做一点特殊处理。
  // 先将其乘上一个2^54，这样它就一定变成了一个规格化的数。
  // 我们之后会获取这个数的阶码，因为这里我们乘上了一个2^54
  // 这里的k要预先减去54.
  k -= 54; x *= two54; /* subnormal number, scale up x */
  hx = __HI(x);		/* high word of x */
} 
```

接下来处理非规格化数的第二种情况，如果阶码全为1，那么如果x是无穷，自然我们也返回无穷，如果x是非数NaN，自然我们也返回NaN，glibc中把这两种情况统一了起来，写成了如下的形式。

```c
  if (hx >= 0x7ff00000) return x+x;
```

接下来的代码是第一个难点，也是glibc实现中的第一个细节。

我们先贴代码，然后再解释：

```c
k += (hx>>20)-1023;
hx &= 0x000fffff;
i = (hx+0x95f64)&0x100000;
__HI(x) = hx|(i^0x3ff00000);	/* normalize x or x/2 */
k += (i>>20);
f = x-1.0;
```

再回顾一下思路篇的内容。浮点数在计算机中的表示是`1.xxxx * 2^{exp}`​，我们要先计算这个`ln 1.xxxx`​，但是问题是，假如尾数也比较大，譬如说是`1.9999`​，那么这样直接用公式算好像还是有一点大，glibc是这么做的，就看这个`1.xxx`​（glibc的文档里面用的是`1+f`​，我们后面也用`1+f`​好了），跟`sqrt(2)`​做对比，如果这个`1+f`​的值超过了`sqrt(2)`​，那么这个把这个数除以2。下面的代码其实就是这个意思，不过其实现还是有一点精巧的。我们一行一行地来看。

```c
// hx是x高32位的十六进制形式，它的高2-12位就是阶码位，右移20位再减去1023就是阶码
// 也就是exp的值。（不懂的请再阅读一下IEE754标准）
k += (hx>>20)-1023;
// 然后我们就来看这个1+f，我们主要关注的是这个f，IEEE754告诉我们，这个尾数f的十六进制
// 就是把高12位变成0即可得到，下面的代码运行后，hx的值和lx的值合在一起就是尾数f的十六进制
hx &= 0x000fffff;
```

接下来的代码是第一个glibc令人非常疑惑的点：

```c
i = (hx+0x95f64)&0x100000;
```

我们详细解释下。首先`sqrt(2)`​的值，十进制是1.414213...，而它的十六进制表示，在64位的计算机上，它的高32位是`0x3ff6a09c`​，它的前32的尾数部分呢，就是`0x3ff6a09c & 0x000fffff=0x6a09c`​，对于我们现在所拥有的尾数`f`​而言，我们怎么判断这个`1+f`​是否超过了`sqrt(2)`​呢？那么我们就是看这个`f`​的值有没有超过这个`(1<<20) - 0x6a09c = 0x95f64`​。我们就把`f`​给加上了一个`0x95f64`​，然后看加上这个数之后，有没有发生进位，也就是与`0x100000`​做了一个与运算，如果进位了，`i`​的值就是`0x100000`​，如果没有，那么就是`0`​。

接下来的代码：

```c
// i要么是0x00100000要么是0x0
// 如果前面发生了进位，那么i就是0x00100000
// 那么i跟0x3ff00000做异或，结果是0x3fe00000
// 再跟ix做或运算，其实就相当于原先的1.xxxx
// 现在变成了1.xxx * 2 ^{exp-1}
// 就是 1+f现在变成 (1+f)/2了
// 如果前面没发生进位呢？那么就保持原值
// 但是这里稍微注意一下，我们这里把x的值改掉了，
// 这一步做完之后，我们的x其实就只剩下1+f了，要想还原成原先的x，
// 只需要利用k，也就是阶码即可。
__HI(x) = hx|(i^0x3ff00000);	/* normalize x or x/2 */
// 调整一下阶码，如果前面发生了进位，这里的k要加1。否则我们无法还原原先的值。
k += (i>>20);
// 最后，我们获取这个f的十进制值，用现在的x-1即可，因为现在的x = 1+f
f = x-1.0;
```

再来看下面的代码：

```c
// 注意这里hx与lx合起来才是f的值
// 如果f非常小，f < 2^(-20)的话，其实没有必要使用思路篇中讲到的公式
// 直接利用ln(1+f)的展开式子就好了。
// 要进入下面这个if，其实就只有一种可能，就是hx=0
if((0x000fffff&(2+hx))<3) {	/* |f| < 2**-20 */
  // 但是hx=0，lx呢？如果lx也为0，那么f就是0，也就是说问题变成了ln(1 * 2^{exp})
  if(f==zero){
    // 如果k也是0，也就是exp为0的话，那么就是ln 1，此时直接return 0
    if(k==0) 
      return zero;  
    // 如果k不是0，那么就是ln 2^{exp} = exp * ln 2
    // 就是k × ln2
    else {
      // k是一个整数，我们强制类型转换为double，就是dk
      dk=(double)k;
      // 注意ln2_hi + ln2_lo = ln2
      // 因此dk × ln2 = dk × ln2_hi + dk × ln2_lo
      return dk*ln2_hi+dk*ln2_lo;
    }
  }
  // 如果f不是0的话，那么就得利用ln(1+f)的泰勒展式了，我们这里正好有f的值
  // ln(1+f) = f - (0.5f^2 - 1/3 f^3)
  // 这里的R就是计算上面的第二项
  R = f*f*(0.5-0.33333333333333333*f);
  // 如果k为0，那么就直接是ln(1+f)，用f-R即可得到
  if(k==0) return f-R;
  // 如果k不为0，那么就是
  // ln( (1+f) × 2^{exp} ) 
  //    = ln(1+f) + exp × ln2
  //    = f - R + dk*ln2_hi + dk*ln2_lo
  //    = dk*ln2_hi-((R-dk*ln2_lo)-f)
  // 为什么要这么麻烦地改变上面的运算顺序?
  // 是因为一个运算技巧，就是高精度的数与高精度的数优先计算
  // 然后计算低精度的数。
  else {
    dk=(double)k;
    return dk*ln2_hi-((R-dk*ln2_lo)-f);
  }
}
```

再来分析后面：

我们再来回顾一下公式：

$$
\begin{aligned}
\ln x &= 2\big[ (\frac{x-1}{x+1})+\frac{1}{3}(\frac{x-1}{x+1})^3+\frac{1}{5}(\frac{x-1}{x+1})^5+\dots\big]\\
&= 2\sum_{k=0}^{\infty}\frac{1}{2k+1}(\frac{x-1}{x+1})^{2k+1}
\end{aligned}
$$

上面的公式，如果我们把`x`​换成`1+f`​可以得到：

$$
\begin{aligned}
\ln (1+f) &= 2\big[ (\frac{f}{2+f})+\frac{1}{3}(\frac{f}{2+f})^3+\frac{1}{5}(\frac{f}{2+f})^5+\dots\big]\\
&= 2\sum_{k=0}^{\infty}\frac{1}{2k+1}(\frac{f}{2+f})^{2k+1}
\end{aligned}
$$

为了计算方便，我们将上面的公式展开到k为7的项，然后改写成下面的形式：

$$
\begin{aligned}
s &= \frac{f}{2+f} \\
z &= s^2 \\
w &= z^2 \\
\ln (1+f) &= s(2 + \frac{2}{3}z + \frac{2}{5}w +\frac{2}{7}zw +\frac{2}{9}w^2 + \frac{2}{11}zw^2+\frac{2}{13}w^3 + \frac{2}{15}zw^3) \\
t1 &=   \frac{2}{5}w + \frac{2}{9}w^2 + \frac{2}{13}w^3 \\
t2 &= \frac{2}{3}z + \frac{2}{7}zw +  \frac{2}{11}zw^2 + \frac{2}{15}zw^3 \\
&=z(\frac{2}{3} + \frac{2}{7}w +  \frac{2}{11}w^2 + \frac{2}{15}w^3)
\end{aligned}
$$

下面的代码就是上面的公式的实现。（请暂时忽略掉`i`​和`j`​）。最终得到的`R`​是一个余项，确切地说是`sR = 2s-ln(1+f)`​

```c
  s = f/(2.0+f); 
  dk = (double)k;
  z = s*s;
  i = hx-0x6147a;
  w = z*z;
  j = 0x6b851-hx;
  t1= w*(Lg2+w*(Lg4+w*Lg6)); 
  t2= z*(Lg1+w*(Lg3+w*(Lg5+w*Lg7))); 
  i |= j;
  R = t2+t1;
```

然后我们来解释一下上面代码中的`i`​和`j`​，最终`i`​的值是`(hx - 0x6147a) | (0x6b851 - hx)`​，这个值具体是什么不重要，我们看到后面一行的`if(i <0)`​，就知道这里最重要的是`i`​的符号，那么也就是说，`hx`​只有在`0x6147a`​到`0x6b851`​这个区间内，才有`i>0`​，如果不在这个范围内，`i<0`​。

我们先继续往下看：

```c
  if(i>0) {
    hfsq=0.5*f*f;
    if(k==0) return f-(hfsq-s*(hfsq+R));
    else return dk*ln2_hi-((hfsq-(s*(hfsq+R)+dk*ln2_lo))-f);
  } else {
    if(k==0) return f-s*(f-R);
    else return dk*ln2_hi-((s*(f-R)-dk*ln2_lo)-f);
  }
```

前面已经提到了，我们现在得到了一个余项`R`​，那么要计算这个`ln(1+f)`​，只需要`2s-sR`​即可。不过这样精度可能还是有一点问题，因为这里是减法，高精度数与高精度数之间的减法可能会导致精度损失，因此glibc把`2s`​这个高精度数，转换成一个高精度数与低精度数的组合，将低精度数与高精度数做减法，可以相当程度地保留精度。（这一块技巧笔者也正在学习中）。

glibc中做了两种变换，一种是：

$$
2s = \frac{2f}{2+f} = f - sf
$$

这一种适用于`i<0`​的情况，也就是更一般的情况。

另一种是：

$$
2s = \frac{2f}{2+f} = f - (h - sh) \text{  where  } h = \frac{1}{2}f^2
$$

这一种精度会更高。（根据glibc的注释）

这一种适用与`i>0`​的情况，此时`0x6147a<hx<0x6b851`​，考虑到`hx`​与`lx`​组合起来就是`f`​，那么看`1+f`​的值，也就是`0x3ff0000 | hx`​就是`0x3ff6147a~0x3ff6b851`​，也就是1.38到1.42，换句话说也就是在`sqrt(2)`​附近。（为什么x的值在`sqrt(2)`​附近就要用更高精度的算法，笔者也暂时不太明白。）

然后我们就可以来进一步分析代码了。

```c
// 如果x的值位于sqrt(2)附近，使用更高精度的算法
if(i>0) {
  hfsq=0.5*f*f;
  // 2s = f - h + sh
  // ln(1+f) = 2s - sR = f - h + sh - SR
  //         = f - (h - s (h+R))
  if(k==0) return f-(hfsq-s*(hfsq+R));
  // 当然，如果阶码不为0，就要加上一个k × ln2 = k × ln2_hi + k × ln2_lo
  // 注意调整顺序使得有更高的精度
  else return dk*ln2_hi-((hfsq-(s*(hfsq+R)+dk*ln2_lo))-f);
} else {
  // 对于更一般的情况，使用一般算法即可
  if(k==0) return f-s*(f-R);
  // 仍然是与上面同样的逻辑。
  else return dk*ln2_hi-((s*(f-R)-dk*ln2_lo)-f);
}
```
