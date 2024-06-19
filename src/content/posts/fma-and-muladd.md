---
title: 近距离观察fma对精度的提升
description: fma即融合乘加，也就是对三个数，前两个数先乘，再与第三个数相加。很多人不理解，为什么fma与普通的先乘后加有精度上的差异，这里做一个细致的观察。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Math/5.jpg
category: 技术原理
published: 2023-05-26
tags: [Math]
---

## Overview

fma是乘法融合加法指令，对于三个浮点数$a, b, c$而言，数学表达式为$a\times b + c$，但是在计算机中，我们不能用一个准确的值来表示一个实数，只能用有限精度的位来表示一个数，这么一来，一个fma如果不扩大精度，直接代之以乘法然后加法的话，精度可能会损失。但是有些机器上又可能既没有扩大精度的手段，也没有fma的指令，如此一来，在一些对精度有要求的场合中，就需要探索一些既不用扩大精度，也不用fma指令，单纯使用软件算法来提高精度。尽管可以确信单次fma与mul+add联合的误差ulp只有1，但是在数学函数的开发中，因为使用多项式来逼近一些数学函数，就需要使用多次fma，从而就会有误差越拉越大的问题。本博客首先研究了误差的起源，然后给出了fma实现的简洁算法，并且给出了参考文献来论证算法的正确性。最后给出了相应的C代码。

## 除法与倒数+乘法的误差问题

我们先来抛开fma，来思考一个问题，除法与倒数再乘法的区别。这里我们用2.4除3的例子，看如下的C代码：

```c
#include <stdio.h>

// statement expression
// $F(0x3eaaaaab) equals to 1/3
// suppose a = 3.0, $I(a) is equivalent to the integer 0x40400000
#define $F(N) ({ unsigned C = N; *(float*)&C; })  // 注1
#define $I(N) *(int*)&N

int main() {
  float a = 2.4f;
  float a_div_3 = a / 3.0f;
  float a_mul_recip_3 = a * $F(0x3eaaaaab);  // 1/3
  printf("a = %f(0x%x), a/3 = %f(0x%x), a*(1/3) = %f(0x%x)\n",
          a, $I(a), a_div_3, $I(a_div_3), a_mul_recip_3, $I(a_mul_recip_3));
  return 0;
}
```

> 注1：$F(N) 用于直接填写十六进制来代表一个浮点数。

上面的运行结果为：

```plaintext
a = 2.400000(0x4019999a), a/3 = 0.800000(0x3f4ccccd), a*(1/3) = 0.800000(0x3f4cccce)
```

从上面的代码中可以看出来，倒数再乘，与直接除有直接的精度差异。这个也是一些asic架构，如果没有除法指令，就会面临的问题之一。

之所以造成这种差异，很重要的点在于上面`1/3`的精度不足，这里有两种做法。

**如果架构允许我们扩大精度**

那么我们可以直接使用double，下面的代码中C编译器会直接使用双精度乘法。

```c
float a_mul_recip_3 = a * (1.0/3.0);
```

**如果架构不允许我们扩大精度**

我们其实仍然有办法，我们可以预先把`1/3`拆分成两个数，然后与除数分别相乘。也就是说改成如下的形式：

```c
// $F(0x3eaaaaa8) + $F(0x33aaaaab) 实际上比$F(0x3eaaaaab)表示的1/3有更高的精度
float a_mul_recip_3 = a * $F(0x3eaaaaa8) + a * $F(0x33aaaaab);
```

这里`$F(0x3eaaaaa8) + $F(0x33aaaaab)`实际上比`$F(0x3eaaaaab)`表示的`1/3`有更高的精度，因此可以更为准确地计算`a/3`的值。

那么这里要引入的一个重要的结论就是，**如果架构不允许我们提高精度，那么我们可以通过拆数来提高运算精度**，上面的两个数实际上是通过`double`来进行计算得到，代码如下：

```c
double d3 = 1.0 / 3.0;
long d3_hi_l = *(long*)&d3 & 0xffffffff00000000UL;
double d3_hi_d = *(double*)&d3_hi_l;
double d3_lo_d = d3 - d3_hi_d;
float d3_hi = (float)d3_hi_d;
float d3_lo = (float)d3_lo_d;
printf("d3_hi = %f(0x%x), d3_lo = %f(0x%x)\n", d3_hi, $I(d3_hi), d3_lo, $I(d3_lo));
```

上面可以得到：

```plaintext
d3_hi = 0.333333(0x3eaaaaa8), d3_lo = 0.000000(0x33aaaaab)
```

## fma的误差问题

我们来看下面的例子，这个例子稍显极端，不过很能说明问题：

```c
#include <math.h>
#include <stdio.h>

#define $F(N) ({ unsigned C = N; *(float*)&C; })
#define $I(N) *(int*)&N

int main() {
  float a = $F(0x3fa2ffff);   // 大约为1.273437
  float b = $F(0x3fa2ffff);   // 重点是后8字节为ffff
  float c = 0.009f;
  float fma = fmaf(a, b, c); 
  float muladd = a*b+c;
  printf("fma = %f(0x%x), muladd = %f(0x%x)\n",
          fma, $I(fma), muladd, $I(muladd));
  return 0;
}
```

上面代码的运行结果为：

```plaintext
fma = 1.630643(0x3fd0b8e7), muladd = 1.630643(0x3fd0b8e6)
```

这里就发现了有些许的误差，ulp=1。

那么如何解决这个问题：

当然，**如果硬件允许我们使用更高精度**，那么我们自然可以使用：

```c
float muladd = (double)a*b+c;
```

这样与fma的结果可以保持一致。

如果硬件不允许我们提高精度，我们可以来探究一下误差产生的原因。问题主要出在`a*b`上，在双精度乘法下，`a*b`的十六进制表示为：`0x3ff9f23fae800040`，但是如果使用单精度乘法，那么`a*b`的结果拓展成`double`后就是`0x3ff9f23fa0000000`，因为`a*b`出现了舍入，造成`a*b`的值稍微小了一点，因此在后面`+c`的时候，结果会比`fmaf`稍微小一点。

另外也可以推测，这其实是单次fma的最大误差，**如果在代码中只使用了1次fma，那么ulp=1实际上还尚可以接受**，不幸的是，我们的数学函数通常是用多项式来逼近，这样一来就需要连着使用多个fma，如此一来，就有可能产生误差传递的问题，导致最后结果的误差变得越来越大，因此有的时候，即便是ulp=1的误差可能也无法接受，因此还是需要探索提高精度的办法。

既然我们已经搞清楚误差产生的原因是因为单精度数产生的舍入误差，那么我们就可以按照前一个小结所给的思路，将整个数进行拆分，相乘，再相加。

## 不使用更高精度的fma

我们上述的例子比较极端，其极端之处就在于如下两行：

```c
  float a = $F(0x3fa2ffff);
  float b = $F(0x3fa2ffff);
```

`a`和`b`的后两个字节都是`ffff`，这种条件下，几乎必然有`(double)(float)a*b < (double)a*b`。由此会导致`a*b+c <= fmaf(a, b, c)`。

那么我们可以进行拆数，再相加的技巧：

```c
float __fmaf(float a, float b, float c) {
  float ahi = $F($I(a) & 0xfffff000);
  float bhi = $F($I(b) & 0xfffff000);
  float alo = a - ahi, blo = b - bhi;
  return alo*blo + c + ahi*blo + bhi*alo + ahi*bhi;
}
```

利用上面的函数来代替原先的`a*b+c`，就可以得到更为准确的结果。

```c
#include <math.h>
#include <stdio.h>

#define $F(N) ({ unsigned C = N; *(float*)&C; })
#define $I(N) *(int*)&N

int main() {
  float a = $F(0x3fa2ffff);   // 大约为1.273437
  float b = $F(0x3fa2ffff);   // 重点是后8字节为ffff
  float c = 0.009f;
  float fma = fmaf(a, b, c); 
  float __fma = __fmaf(a, b, c);
  printf("fma = %f(0x%x), __fma = %f(0x%x)\n",
          fma, $I(fma), __fma, $I(__fma));
  return 0;
}
```

可以得到结果：

```plaintext
fma = 1.630643(0x3fd0b8e7), __fma = 1.630643(0x3fd0b8e7)
```

可以看出，的确提高了精度。

## 理论验证

这一算法的正确性来源于2020年5月的一篇论文：

[https://hal.archives-ouvertes.fr/hal-02470782v2/document](https://hal.archives-ouvertes.fr/hal-02470782v2/document)

论证了这种算法的正确性。

但是，要指出的是，我们上面的代码，很遗憾，并不是通用的`fma`算法，这篇论文指出，如果可以做到严格地按照数量级从小到大的顺序相加，最终的结果才是正确的，与fma一致。例如将上面代码的修改为：

```c
float __fmaf(float a, float b, float c) {
  float ahi = $F($I(a) & 0xfffff000);
  float bhi = $F($I(b) & 0xfffff000);
  float alo = a - ahi, blo = b - bhi;
  return ahi*bhi + ahi*blo + bhi*alo + alo*blo + c;  // 相加的顺序改变
}

int main() {
  float a = $F(0x50a2ffff);   // a和b的值调大
  float b = $F(0x50a2ffff);
  float c = 0.009f;
  float fma = fmaf(a, b, c);
  float __fma = __fmaf(a, b, c);
  printf("fma = %f(0x%x), __fma = %f(0x%x)\n",
          fma, $I(fma), __fma, $I(__fma));
  return 0;
}
```

会发现结果还是差了一点：

```plaintext
fma = 478624448445310566400.000000(0x61cf91fd),
__fma = 478624483629682655232.000000(0x61cf91fe)
```

但是如果使用原来的相加顺序就没有问题，结果为：

```plaintext
fma = 478624448445310566400.000000(0x61cf91fd),
__fma = 478624448445310566400.000000(0x61cf91fd)
```

因而，在开发过程中，我们可以有两种选择，一是开发通用的fma，但是通用的fma因为考虑的因素可能过多，因此速度上可能不尽如人易。二是，既然我们通常是用于数学函数开发中，并且我们多半已经知道多项式的系数值，也就是说我们已经事先知道拆数后项与项之间的数量级关系，我们自然可以预判相加顺序，直接使用以上最简洁的算法就好。

