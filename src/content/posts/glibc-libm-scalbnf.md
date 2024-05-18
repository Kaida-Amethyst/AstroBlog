---
title: glibc源码剖析 - scalbnf
description: 剖析一下glibc中scalbnf的实现，熟悉IEEE 754标准才能解读好这个代码
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Math/3.jpg
category: 技术原理
published: 2022-03-14
tags: [C, Math]
---

----

## Function Prototype

​`scalbn`​函数定义在`camth`​库中，声明如下：

```cpp
double scalbn(double x, int n);
```

它的作用是将指定数`x`​缩放$2^n$，相当于：

```cpp
double scalbn(double x, int n) {
    return x*std::pow(2, n)
}
```

不过，上面的做法是非常低效的，而且glibc里面实际上pow函数会调用这个`scalbn`​，如此一来会造成隐式的递归。因此就不能这么做。

---

## Thinking

### 一般情形

考虑到浮点数本身的表示就是$x\times 2^n$的形式，因此将n的加到阶码上即可。但是，很显然需要考虑一些特殊场景。

### 特殊情形

1. $x$本身是0，那么不管$n$是多少，都应该输出0。
2. $x$是无穷大，那么不管$n$是多少，都应该输出对应符号的无穷大。
3. $x$的阶码加上$n$超过了阶码的表示范围，此时应该输出无穷大。
4. $x$的阶码加上$n$正好等于阶码的最大值，此时应该输出无穷大。
5. $x$是非数$\rm{Nan}$，那么不管是多少，都应该输出对应符号的非数$\rm{Nan}$。
6. $x$的阶码加上$n$是一个负数，此时的结果是一个非规格数，有两种情况，一种是0，一种是非规格数的小数。
7. $x$本身是一个非规格化的小数。

上面6,7两点会略微难以理解，实际上也是glibc中scalbn中造成代码理解困难的其中一个原因。如果用C++的`numeric_limits`​来获取double的最小值，大致的数值是$2.225\times 10^{-308}$但是，这是规格化浮点数的最小值，实际上如果把范围再扩大一点，允许阶码是0，那么它的非零最小值（阶码为0，尾数不为0）实际上大致是$4.94\times 10^{-324}$，第6点说得是，可能会计算出一个这样的值，第7点说得是$x$本身是这样的值。

## Source Code

```cpp
#include <iostream>
#include <cmath>
#include <iomanip>
#include <limits>

// 下面的宏要在机器是小端方式才成立，如果机器是大端方式，那么两者是反过来的
// 先把x的地址取出，将其标记成一个int的指针，这里的x原先是double指针，
// 是int的两倍，转为int指针后，如果是小端方式，那么字节是按照逆序排列的，
// 因此在double x的高位在内存中放在低位。double x的低位在内存中放在高位
// 1加上一个int类型指针，是指向了下一个int。
#define __HI(x) *(1+(int*)&x)
#define __LO(x) *(int*)&x

double fdscalbn (double x, int n) {
    // 这里的two54是2^54，twom54是2^-54，其作用在后面介绍。
    double two54   =  1.80143985094819840000e+16; /* 0x43500000, 0x00000000 */
    double twom54  =  5.55111512312578270212e-17; /* 0x3C900000, 0x00000000 */
    // huge表示一个非常大的数，tiny表示一个非常小的数，它俩的具体值其实不重要，
    // 因为这两个实际上是用C来生成无穷大和0的，后面可以看到
    // 用huge*copysign(huge, x)生成了与x相同符号的无穷大，
    // 用tiny*copysign(tiny, x)生成了与y相同符号的0（因为有+0和-0）
    double huge   = 1.0e+300;
    double tiny   = 1.0e-300;
    // k用来存放阶码，hx是x的高位字节，lx是x的低位字节
    int  k,hx,lx;
    // 利用上面的两个宏
    hx = __HI(x);
    lx = __LO(x);
    // 一个位运算来提取x的阶码
    k = (hx&0x7ff00000)>>20;		/* extract exponent */
    // 提出来得k有几种情况
    // 1. k==0，此时x是一个非规格化的数，此时如果尾数位是0，那么x是0，此时返回x
    //    如果尾数位不为0，此时是一个非规格小数，具体处理方法在下面的代码里。
    // 2. k==0x7ff，也就是11个阶码位全为1，此时x可能是无穷大或者nan，如果是
    //    无穷大，那么返回x，如果是nan，返回的不是x！而是一个与x同符号的标准nan
    // 3. 其它情况，也就是x是一个普通的规格数。
    if (k==0) {				/* 0 or subnormal x */
        // k==0，检查尾数位是否为0，如果是的话，返回x。
        if ((lx|(hx&0x7fffffff))==0) return x; /* +-0 */
        // 如果尾数位不是0，它不能正常地在后面进行处理，这里的做法是先将其
        // 乘上2^54，使其先成为一个规格化小数，其实相当于将x的阶码加上一个54
        // 为什么是54，这是因为double的尾数位是52，如果k==0，那么此时n其实还可
        // 以更小，如果n不小于-52，那么计算出来的值仍然可能是一个非规格化小数，
        // 而不是0，如果n小于-52，那么计算出的值应该直接就是对应符号0。
        // 为了稍微多一点灵活性，采用了54，稍稍比52大一点。
        x *= two54; 
        hx = __HI(x);
        k = ((hx&0x7ff00000)>>20) - 54; 
        // 以上的两步，x会变成一个规格化的小数，但是k仍然是0。
        // n过小的情况下，直接用tiny*x直接得到对应符号0
        if (n< -50000) return tiny*x; 	/*underflow*/
    }
    // 如果k==0x7ff，那么返回nan或inf，如果是inf，的确可以直接返回，但是
    // 主要的注意点在于nan，要返回一个标准nan
    // 也就是 0x7fffffffffffffff，或者0xffffffffffffffff，如果直接返回x，
    // 可能会导致返回的是一个非标准的nan。
    if (k==0x7ff) return x+x;		/* NaN or Inf */
    // 让k += n
    k = k+n;
    // k+=n后，又有几种情况：
    // k>=0x7ff，此时应当返回无穷大，但是稍微注意一个额外的问题，就是n可能特别大
    // 会导致k+=n后变成一个负数，这种情况是在后面进行处理的。
    if (k >  0x7fe) return huge*std::copysign(huge,x); /* overflow  */
    // 如果k就是普通地大于0，而没有超过7ff的话，那么就是最一般的情况，就是把k的值放到
    // x的阶码位上就好了。
    if (k > 0) 				/* normal result */
        {__HI(x) = (hx&0x800fffff)|(k<<20); return x;}
    // k<=-54，有两种情况，一种是n过大，大到超过了int的正数界限，此时要判断n是不是一个大数
    // n是大数，返回无穷大。
    // 如果不是因为这种情况才使得k<=-54的话，那么应该返回的是对应符号0。
    if (k <= -54)
        if (n > 50000) 	/* in case integer overflow in n+k */
            return huge*std::copysign(huge,x);	/*overflow*/
        else return tiny*std::copysign(tiny,x); 	/*underflow*/
    // 最后一种情况是k<0，但是没有到-54
    // 此时应该返回一个非规格小数，将阶码位置为0，然后尾数位右移|k|位
    // 这里的做法是，首先将k+=54，使其变成一个规格化小数，例如k=-20，k+=54后为k=34
    // 然后将其放入x中，这样得到的新的x的值会放大2^54
    // 最后再乘上一个2^-54，数值上回到原本的值
    k += 54;				/* subnormal result */
    __HI(x) = (hx&0x800fffff)|(k<<20);
    return x*twom54;
}
```

‍
