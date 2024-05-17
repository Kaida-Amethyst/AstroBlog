---
title: 近距离观察C++20`[[likely]]`和`[[unlikely]]`属性
description: C++20引入了两个新的属性`[[likely]]`和`[[unlikely]]`，这两个属性可以帮助编译器优化代码，这一篇来观察一下对应的汇编代码。
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Cpp/3.jpg
category: Programming
published: 2023-04-02
tags: [C++, Assembly]
---

C++20提供了两个属性：`[[likely]]`和`[[unlikely]]`，用在为编译器提供分支的信息。在更有可能触发的分支上使用`[[likely]]`关键字可以提升一定的程序运行效率。

> 注：尽管这两个属性在C++20中被添加，但是如果使用`std=c++11`的选项，在最新版的C编译器中也是起作用的。

这一节来近距离观察编译的效果。

## Sample Code

```cpp
int foo(int a, int b);

int bar(int a, int b, int c);

int main(int argc, char *argv[]) {
  if (argc < 5) {
    foo(1, 2);
  } else {
    bar(41, 96, 83);
  }
  return 0;
}
```

这里的`main`函数，我们从标准输入中获取参数个数，如果参数个数小于5，调用`foo`这个函数，否则调用`bar`这个函数。

然后，我们在两个分支上，分别添加`[[liekly]]`和`[[unlikely]]`属性。

## First Case

第一次尝试，我们把`[[likely]]`添加到一个分支上：

```cpp
int main(int argc, char *argv[]) {
  if (argc < 5) [[likely]] {
    foo(1, 2);
  } else [[unlikely]]  {
    bar(41, 96, 83);
  }
  return 0;
}
```

接着使用`clang++`编译，注意这里使用`-O1`，`-O0`编译器不会使用优化。打印得到的x86汇编代码，为了节省篇幅，这里把编译出来的冗余信息进行了删除。

```x86asm
main:
	pushq	%rax
	cmpl	$4, %edi
	jg	.LBB0_2
	movl	$1, %edi
	movl	$2, %esi
	callq	_Z26fooii@PLT
	popq	%rcx
	retq
.LBB0_2:
	movl	$41, %edi
	movl	$96, %esi
	movl	$83, %edx
	callq	_Z29bariii@PLT
	popq	%rcx
	retq
```

## Second Case

第二次尝试，我们把`[[likely]]`加到第二个分支上：

```cpp
int main(int argc, char *argv[]) {
  if (argc < 5) [[unlikely]] {
    foo(1, 2);
  } else [[likely]] {
    bar(41, 96, 83);
  }
  return 0;
}
```

然后再进行编译：

```x86asm
main:
	pushq	%rax
	cmpl	$4, %edi
	jle	.LBB0_1
	movl	$41, %edi
	movl	$96, %esi
	movl	$83, %edx
	callq	_Z29bariii@PLT
	popq	%rcx
	retq
.LBB0_1:
	movl	$1, %edi
	movl	$2, %esi
	callq	_Z26fooii@PLT
	popq	%rcx
	retq
```

## Compare two assembly

比较一下这两块代码：

```x86asm
main:
	pushq	%rax
	cmpl	$4, %edi
	jg	.LBB0_2
	movl	$1, %edi
	movl	$2, %esi
	callq	_Z26fooii@PLT
	popq	%rcx
	retq
.LBB0_2:
	movl	$41, %edi
	movl	$96, %esi
	movl	$83, %edx
	callq	_Z29bariii@PLT
	popq	%rcx
	retq
```

```x86asm
main:
	pushq	%rax
	cmpl	$4, %edi
	jle	.LBB0_1
	movl	$41, %edi
	movl	$96, %esi
	movl	$83, %edx
	callq	_Z29barii@PLT
	popq	%rcx
	retq
.LBB0_1:
	movl	$1, %edi
	movl	$2, %esi
	callq	_Z26fooii@PLT
	popq	%rcx
	retq
```

从这里可以看到，第一种情况，如果为`argc < 5`的情形标注`[[likely]]`，那么函数的跳转会把调用`bar`的代码安排在“稍微远一点”的地方。如果反过来为`argc < 5`的情形标注`[[unlikely]]`，那么编译器会把调用`foo`的代码往后安排。把更可能的分支安排在接近`cmp`指令的地方，从而避免频繁地跳转对性能的干扰。
