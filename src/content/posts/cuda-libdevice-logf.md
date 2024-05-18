---
title: cuda libdevice中的logf函数
description: glibc的logf虽然比较精准，但不太适合在gpu这种向量化架构上跑，看看cuda是怎么计算的
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Math/5.jpg
category: 技术原理
published: 2022-08-17
tags: [C++]
---

----

cuda计算log函数与glibc的实现方法大不相同，这里对cuda实现log的源码做一个分析。有关glibc-log的源码解析，参考前面的博客。

## Function Prototype

```c
__device__ float __logf (float x)
```

## Ptx汇编

我们把cuda中logf函数的ptx汇编打印出来，如下：

```cuda
	ld.param.f32 	%f5, [_Z9devKernelf_param_0];

	mul.f32 	%f6, %f5, 0f4B000000;
	setp.lt.f32	%p1, %f5, 0f00800000;
	selp.f32	%f1, %f6, %f5, %p1;
	selp.f32	%f7, 0fC1B80000, 0f00000000, %p1;
	mov.b32 	 %r1, %f1;
	add.s32 	%r2, %r1, -1059760811;
	and.b32  	%r3, %r2, -8388608;
	sub.s32 	%r4, %r1, %r3;
	mov.b32 	 %f8, %r4;
	cvt.rn.f32.s32	%f9, %r3;
	mov.f32 	%f10, 0f34000000;
	fma.rn.f32 	%f11, %f9, %f10, %f7;
	add.f32 	%f12, %f8, 0fBF800000;
	mov.f32 	%f13, 0f3E1039F6;
	mov.f32 	%f14, 0fBE055027;
	fma.rn.f32 	%f15, %f14, %f12, %f13;
	mov.f32 	%f16, 0fBDF8CDCC;
	fma.rn.f32 	%f17, %f15, %f12, %f16;
	mov.f32 	%f18, 0f3E0F2955;
	fma.rn.f32 	%f19, %f17, %f12, %f18;
	mov.f32 	%f20, 0fBE2AD8B9;
	fma.rn.f32 	%f21, %f19, %f12, %f20;
	mov.f32 	%f22, 0f3E4CED0B;
	fma.rn.f32 	%f23, %f21, %f12, %f22;
	mov.f32 	%f24, 0fBE7FFF22;
	fma.rn.f32 	%f25, %f23, %f12, %f24;
	mov.f32 	%f26, 0f3EAAAA78;
	fma.rn.f32 	%f27, %f25, %f12, %f26;
	mov.f32 	%f28, 0fBF000000;
	fma.rn.f32 	%f29, %f27, %f12, %f28;
	mul.f32 	%f30, %f12, %f29;
	fma.rn.f32 	%f31, %f30, %f12, %f12;
	mov.f32 	%f32, 0f3F317218;
```

上面的`[_Z9devKernelf_param_0]`​，就是参数x。

上面的ptx汇编看起来多少有一些麻烦，我们把上面的汇编依据逻辑改写成如下的C代码：

```c
float cudalogf(float x) {
  float k, f, e, s, t, r;

  int ix = *(int*)&x;
  k = 0;
  if (ix < 0x00800000) {
    x = x * 8388608.0f;
    k = -23.0f;
  }
  
  ix = *(int*)&x;
  int tmp_v1 = (ix - 0x3f2aaaab) & 0xff800000;
  int tmp_v2 = ix - r3;
  e = (float)tmp_v1;
  f = *(float*)&tmp_v2;

  k = fmaf(e, 1.19209290e-7f, k);
  f = f - 1.0f;
  s = f * f;
  r =             -0.130310059f;  // -0x1.0ae000p-3
  t =              0.140869141f;  //  0x1.208000p-3
  r = fmaf (r, s, -0.121483512f); // -0x1.f198b2p-4
  t = fmaf (t, s,  0.139814854f); //  0x1.1e5740p-3
  r = fmaf (r, s, -0.166846126f); // -0x1.55b36cp-3
  t = fmaf (t, s,  0.200120345f); //  0x1.99d8b2p-3
  r = fmaf (r, s, -0.249996200f); // -0x1.fffe02p-3
  r = fmaf (t, f, r);
  r = fmaf (r, f,  0.333331972f); //  0x1.5554fap-2
  r = fmaf (r, f, -0.500000000f); // -0x1.000000p-1  
  r = fmaf (r, s, f);
  r = fmaf (k,  0.693147182f, r); //  0x1.62e430p-1

  return r;
}
```

## 计算思路

大体的思路如下：

求解：

$$
y = \ln{x}
$$

我们知道计算机中的浮点数通常表示为$(1+f)\times 2^{exp}$的形式，把这个形式代入，就得到：

$$
y = \ln{(1+f)\times 2^{exp}} = \ln{(1+f)} + exp \ln 2
$$

所以这里我们就分成两部分求解，第一部分求`ln (1+f)`​，第二部分求`exp × ln 2`​。

cuda与glibc不太一样的点在于，对于第一部分，cuda采用了`ln (1+f)`​的一个直接的泰勒公式，因为这里的`f`​是比较接近0的。对于第二部分，cuda并不是直接计算的`exp × ln 2`​，而是稍作变形，使用了`exp × （1/log_2{e}）`​这个值。

## 源码剖析

我们来一行一行地分析下源码：

```c
int ix = *(int*)&x;
k = 0;
if (ix < 0x00800000) {
  x = x * 8388608.0f;
  k = -23.0f;
}
```

首先要考虑的是输入`x`​是一个非规格小数的问题，如果是一个非规格小数，cuda这里让其乘上一个8388608，其实它是`2^23`​。这么一乘之后，它的值就必定是一个规格化的浮点数了。这个做法与glibc也有差别，通常glibc遇到这种问题会乘以`2^25`​。当然在这种情况下我们的阶码要预先减去23。

然后是第二部分：

```c
// 我们的x可能做了变动，因此我们再次获取一下x的十六进制形式
ix = *(int*)&x;
// x的十六进制与0x3f2aaaab作差，再与0xff800000做与运算
int tmp_v1 = (ix - 0x3f2aaaab) & 0xff800000;
// 得到的值再减去tmp_v2
int tmp_v2 = ix - tmp_v1;
e = (float)tmp_v1;
f = *(float*)&tmp_v2;
```

上面的代码是整个代码中最难懂的一部分，我们先来解释一下`0x3f2aaaab`​的意思。我们要把它看成两个部分，阶码和尾数的加和，阶码部分是`0x3f000000`​，尾数部分是`0x2aaaab`​。我们先不去管尾数部分，假设我们是如下的操作：

```c
(ix - 0x3f00000) & 0xff800000
```

它其实就相当于我们把`x`​的阶码取出后再加1（因为这里是`0x3f000000`​（阶码为126-127=-1），而不是`0x3f800000`​（阶码为0）），其实就相当于我们做了如下的操作：

```c
((ix >> 23) - 126) << 23
```

然后我们再看尾数，明白了上面的过程之后，我们就知道，如果`x`​的尾数如果小于`0x2aaaab`​的话，此时的阶码并不是减`126`​，而是减`127`​，也就是说上面的式子：

```c
((ix >> 23) - 126) << 23  // 如果x的尾数超过了0x2aaaab
((ix >> 23) - 127) << 23  // 如果x的尾数不超过0x2aaaab
```

换句话说，这里的`tmp_v1`​的值，其实相当于下面的代码：

```c
int tmp_v1;
if ( (ix&0x7fffff) > 0x2aaaab) {
  tmp_v1 = (ix >> 23 - 126) << 23;
} else {
  tmp_v1 = (ix >> 23 - 127) << 23;
}
```

然后这里的`tmp_v2`​是`ix-tmp_v1`​，这样一来首先尾数还保存着，其次阶码是126或者127。换而言之，就是此时`tmp_v1`​保存了原先`x`​的阶码信息，`tmp_v2`​是x的`1+f`​的信息。

再来看这里的`0x2aaaab`​的意思，实际上它是与`x`​的`1+f`​做了比较，当`f`​超过了`0x2aaaab`​的时候，阶码会多减一个1。也就是说`1+f`​的值，超过了`0x3f8aaaab`​的时候（注意是`0x3f8aaaab`​），也就是超过了1.333333的时候，阶码会多减一个1。后面代入多项式运算的`(1+f)`​变成了`(1+f)/2`​。

后面的代码就稍微简单了，就是利用`ln(1+f)`​的泰勒多项式。

$$
\ln(1+f) = f - \frac{1}{2}f^2+\frac{1}{3}f^3-\frac{1}{4}f^4+\cdots
$$

```c
  // 这里的1.19209290e-7就是2^{-23}，因为先前的k它的阶码存在了
  // 一个浮点数上，这里乘以一个2^{-23}就可以将其拿出。
  k = fmaf(e, 1.19209290e-7f, k);
  f = f - 1.0f;
  s = f * f;
  r =             -0.130310059f;  // -0x1.0ae000p-3
  t =              0.140869141f;  //  0x1.208000p-3
  r = fmaf (r, s, -0.121483512f); // -0x1.f198b2p-4
  t = fmaf (t, s,  0.139814854f); //  0x1.1e5740p-3
  r = fmaf (r, s, -0.166846126f); // -0x1.55b36cp-3
  t = fmaf (t, s,  0.200120345f); //  0x1.99d8b2p-3
  r = fmaf (r, s, -0.249996200f); // -0x1.fffe02p-3
  r = fmaf (t, f, r);
  r = fmaf (r, f,  0.333331972f); //  0x1.5554fap-2
  r = fmaf (r, f, -0.500000000f); // -0x1.000000p-1  
  r = fmaf (r, s, f);
  
```

求出`ln(1+f)`​，后，还有最后一步，就是加上一个`exp × ln 2`​，这里是利用了`exp × 1/log_2(e)`​。

```c
r = fmaf (k,  0.693147182f, r); //  0x1.62e430p-1，其实就是1 / log_2(e)
```

# 参考链接

[1]: 有关cuda内部计算的一些系数问题，可能需要参考：[https://arxiv.org/abs/1508.03211](https://arxiv.org/abs/1508.03211)
