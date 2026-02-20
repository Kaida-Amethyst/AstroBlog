---
title: MoonLLVM - 小型编译框架的理想
description: 小型编译框架也有大作为
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Java/2.jpg
category: 编程实践
published: 2025-12-19
tags: [LLVM, MoonBit]
---

---

求学期间，我曾经非常迷恋 C++：模板元编程很酷，标准库看起来很「工程」、很「专业」。那会儿，我也用 C++ 写过不少东西：Lua 编译器、C 编译器，还有各种各样的小工具等等。后来毕业以后，又在国内某知名芯片公司，也做过 LLVM / MLIR 相关工作，写过优化 Pass、后端代码生成等。

但当我的经验越来越丰富，见到的东西越来越多之后，我越来越有一种感觉：

> 如果LLVM不是建构在C++上的就好了。

## LLVM -- 杰出的设计

作为知名的编译器后端框架，**LLVM 的架构设计是非常优秀的**。它的 IR 设计、Pass 管线、Def-Use 链，以及对硬件的抽象，都是教科书级别的。也正因如此，LLVM 成了工业界编译器后端的标准。2013年，LLVM被授予了2012ACM软件系统奖，已足以说明工业界对它的认可。

但 LLVM 的基石语言C++，则是另一个极端。

## 问题频出的C++

### 1. 语言太复杂

**我们在“对付语言”上花的精力，可能超过了“对付业务”。** 编译器本身已经是一个超级复杂的系统，但 C++ 又额外叠加了一层复杂度：

- 一个变量可能有二十几种初始化方式
- 一个类可以有足足六种构造函数；
- 模板元编程的各种黑魔法，且与C++主体编程范式大为不同的函数式范式。
- 标准库里大量行为细节、陷阱、烂尾特性和词不达意，例如变长数组叫`vector`，而真正的变长数组`valarray`是一个烂尾的特性；`vector<bool>`并不是`bool`的容器；`std::regex`离谱的性能问题；`std::remove`实际上的作用是把元素移动到末尾等等。
- 各种看起来炫酷，实际上有些“故作高深”的名词，例如`SFINAE`，`CRTP`，`RTTI`，`RAII`，`monostate`等等。

写编译器的工程师，本来应该把大部分时间花在「语言语义、优化算法、后端架构」上，但在 C++ 世界，团队不得不在很长时间里花大量精力，放在给新人培训 C++ 语法和习惯用法，或者是与各种构建配置、ABI、模板错误信息搏斗。

### 2. 字符串处理上问题频出

编译器本质上你可以看做是一个复杂的字符串处理程序，但一个很不幸的事实是： C++ 标准库在字符串处理上的支持并不友好，缺少真正可靠的字符串处理库，常用的`std::string`或者`std::regex`都有不小的问题。这使得现实世界中，很多经典的库可能会选择自己动手制作字符串库，但是C++难以集成第三方库的特点由阻碍了开发者去使用它们。

而这对于想用 C++ 入门写编译器的学生来说就非常不友好了，很多人连第一关「词法分析」可能都过不去。

### 3. 编译慢，调试体验差

C++的编译实在是太慢了，越大的项目这个问题越明显。这是C++本身模板展开和头文件的重复编译导致的这个问题，很多情况下，开发者稍微改一点点，就要重新等很久。

但编译器又是一个「必须频繁读源码、单步调试」的系统软件，因为编译器与Web程序或者游戏程序不同，很多 bug 无法靠打 log 简单定位；需要频繁重编译 + GDB / LLDB 调试。

这使得日常开发体验非常不友好，debug 版 LLVM + C++ 的组合，有时慢到让人怀疑人生。编译和调试的效率底下结果导致的问题就是企业的开发成本非常高。

### 4. 构建系统和第三方依赖太折腾

几乎每个写 C++ 的人都和各种构建系统打过交道，Makefile / Ninja / Bazel / MSBuild / SCons / ...等等。而CMake 虽然是事实标准，但在语法和使用体验上，实在是一言难尽。

构建系统混乱带来的直接后果就是引入第三方库极其麻烦，因为不同的包可能使用了不同的构建工具。即使都使用了CMake，ABI、编译选项、链接方式的兼容性问题也常常出现。一些GitHub Star很高的项目，会推崇单头文件模式，但是这样又会带来编译速度的问题。

构建系统和第三方库的问题，造成了一个C++特色：C++ 程序员，对「重复造轮子」往往已经习以为常。

---

## 如果有一个「非 C++ 的 LLVM 伴侣」……

假如我们有这样一个东西：

- 能和真实的 LLVM 平滑互操作，能够生成兼容 LLVM 工具链的 IR / bytecode；
- 有着非常优秀的字符串处理，ADT，模式匹配语法，而且语法简单，上手容易，AI友好。
- **更轻量、编译速度快、源码更容易读**；

那它是不是可以作为一个很好的：

- 学习编译器的入口与实验场；
- 小型芯片公司 / 小团队的编译后端；
- 以及真实 LLVM 之前的「缓冲层」和「跳板」？

这就是 MoonLLVM 想做的事情。

> MoonLLVM：A Tiny, Friendly Companion to LLVM

GitHub 地址：[moonbitlang/MoonLLVM](https://github.com/moonbitlang/MoonLLVM)

---

## MoonLLVM 的定位：不是「重写 LLVM」，而是「LLVM 的友好伴侣」

先把预期讲清楚：

> **MoonLLVM 不是 LLVM 的重构版**；也**不打算直接取代 LLVM**。

现实是：LLVM 过去二十多年集结了多家大公司的巨大投入，在优化、多后端支持、生态广度上，有压倒性的优势。MoonLLVM 短期内不可能、也没必要在这些维度上正面进攻。

MoonLLVM 想填补的是另一块空白：

- 让**学生和初学者**更容易接触 IR / Pass / 后端；
- 让**小型芯片公司、小团队**可以用更低成本做原型和特定场景的后端；
- 让**熟悉 LLVM 的工程师**多一个更轻量、可退出的选项。

---

## 核心目标：与真 LLVM 的三层互操作

MoonLLVM 和很多「自建小框架」，例如QBE，Cranelift等最大的不同，是我们从一开始就认真设计了与真 LLVM 的互操作，而不是造一个完全封闭的「私有宇宙」。

可以分三层来看：

### 1.1 代码级互操作

- 我们有一个 `llvm.mbt` 包，它是一个真 LLVM 的 MoonBit 绑定，需要本地安装 LLVM。调用`llvm.mbt`得到的IR，就是原版LLVM生成出来的IR。
  github: [https://github.com/moonbitlang/llvm](https://github.com/moonbitlang/llvm)
- MoonLLVM 的 API 与 `llvm.mbt` 有意识地对齐：数据结构、接口设计都保持相似。
- 用户只需改一处：调整 `moon.pkg.json` 配置，即可在 MoonLLVM 和真 LLVM 之间切换：

  设计目标是：MoonLLVM → 真 LLVM 平滑切换；

  - 需要注意的是，反向切回来我们无法保证，这是因为MoonLLVM还是比真LLVM简单太多。
- MoonLLVM / llvm.mbt 的 API 很大程度上参考了原版 C++ LLVM：

  - `Context / Module / Function / BasicBlock / IRBuilder` 等核心概念一一对应；
  - 操作顺序、调用方式尽量保持一致；
  - 对熟悉 C++ LLVM 的工程师来说，几乎不需要换脑子。

### 1.2 模块级互操作（未来）

- MoonLLVM 与 `llvm.mbt` 可以**在同一工程中共存**；
- 提供转换函数，将 MoonLLVM 的中间数据结构**转成** `llvm.mbt` 的数据结构；
- 这意味着：你可以在 MoonLLVM 中做自定义 IR 生成、轻量优化，然后把 IR 交给真 LLVM 做后续优化和后端。

### 1.3 产物级互操作（未来）

- MoonLLVM 生成的 LLVM IR / bytecode：

  - 可以被现有 LLVM 工具链（如 `llc`、`opt`）识别和处理；
  - MoonLLVM 对其能力范围内的 IR 支持 Parse 和再处理。
- 一条典型链路可以是：

  MoonLLVM → LLVM IR → `llvm-opt` 优化 → LLVM IR →（可选）再回 MoonLLVM

从一开始，MoonLLVM 就内建了一个清晰的「退出机制」：

- 用户不用担心「用了 MoonLLVM 以后就被架死在这里」；
- 未来如果有需要：可以逐步把关键路径迁回真 LLVM；或者只在某些阶段用 MoonLLVM 快速试验和开发。

---

## MoonLLVM的其它目标

1. 运行速度：用「适度取舍」换来轻盈

   MoonLLVM 一开始就明确做了一个选择：不追求覆盖所有稀奇古怪的 C 语言特性（比如 VLA 等），C++特性（例如各种C++专用的异常）。也不追求支持所有冷门架构与扩展。

   这样做的好处是：数据结构和算法可以更简单，内部可以大量使用定长整数和更直接的实现，在「生成 IR / 做基础转换」这类场景下，MoonLLVM 在 MoonBit 里调用的整体运行速度，有机会显著快于直接调用真 LLVM。
2. 编译速度快，组件细粒度模块化
   MoonLLVM 有意识地把组件拆得更细，做到**用到哪个编译哪个**，没用到的完全不参与构建；整体编译时间可以维持在较低水平；对开发者来说：改一个 Pass，不需要重编整个「工程」；非常适合做在线 Playground / REPL 的后端；或者是新 Pass / 新后端的实验平台。
3. 无外部依赖：只依赖 MoonBit 工具链
   MoonLLVM 只需要 MoonBit 自身的工具链即可：不需要外部 C++ 编译器；也不需要 CMake / TableGen 等复杂构建工具。对于只会 MoonBit 的开发者来说，安装过程非常简单；
4. 轻量，适合「弱环境」和多种部署形态

   得益于整体设计的轻量化和 MoonBit 语言本身的特性：MoonLLVM 可以跑在性能不算强的 PC 上，也可以通过 WebAssembly 部署到浏览器中，这使得它在部分嵌入式 / 边缘计算环境中也有实际落地的希望；

   这为「MoonBit 在线 Playground」、「嵌入式脚本 / DSL 环境」等场景，提供了很自然的技术路径。

---

## 一个典型的 MoonLLVM 程序

下面这个示例展示了如何用 MoonLLVM 创建一个简单的加法函数：

```moonbit
fn main {
  // 1. 创建上下文与模块
  let ctx = @IR.Context::new()
  let mod = ctx.addModule("my_module")
  let builder = IRBuilder::new()

  // 2. 定义函数类型：i32 add(i32, i32)
  let i32_t = ctx.getInt32Ty()
  let f_type = ctx.getFunctionType(i32_t, [i32_t, i32_t])

  // 3. 向模块中添加函数，并创建入口基本块
  let f = mod.addFunction(f_type, name="add")
  let bb = f.addBasicBlock(name="entry")
  builder.setInsertPoint(bb)

  // 4. 绑定并命名参数
  let x = f.getArg(0).unwrap()
  x.setName("x")
  let y = f.getArg(1).unwrap()
  y.setName("y")

  // 5. 创建加法指令与返回
  let sum = builder.createAdd(x, y, name="sum")
  builder.createRet(sum)

  // 6. 打印生成的 LLVM IR 文本
  println(mod)
}
```

输出的 LLVM IR 大致如下：

```llvm
;; ModuleID = 'demo'
;; Source File = "demo"
;; Generated by MoonLLVM, not llvm

define i32 @add(i32 %x, i32 %y) {
entry:
  %sum = add i32 %x, %y
  ret i32 %sum
}
```

注意顶部的注释部分明确标注了「Generated by MoonLLVM, not llvm」，这与原版 C++ LLVM 的输出可以清晰区分。

这份 IR 既可以交给官方 LLVM 工具链（例如 `llc`）继续编译，也可以作为 MoonMIR 的输入，进一步生成 `riscv64` 或 `aarch64` 汇编。

---

## 对应的 C++ LLVM 程序对比

下面是一段等价的 C++ LLVM API 示例代码，实现同样的 `add(i32, i32) -> i32` 函数：

```cpp
#include "llvm/IR/IRBuilder.h"
#include "llvm/IR/LLVMContext.h"
#include "llvm/IR/Module.h"
#include "llvm/Support/raw_ostream.h"

int main() {
  // 1. 创建上下文与模块
  llvm::LLVMContext ctx;
  auto module = std::make_unique<llvm::Module>("my_module", ctx);
  llvm::IRBuilder<> builder(ctx);

  // 2. 定义函数类型：i32 add(i32, i32)
  llvm::Type *i32_ty = llvm::Type::getInt32Ty(ctx);
  std::vector<llvm::Type *> params(2, i32_ty);
  llvm::FunctionType *fty =
      llvm::FunctionType::get(i32_ty, params, /*isVarArg=*/false);

  // 3. 在模块中创建函数与入口基本块
  llvm::Function *fn =
      llvm::Function::Create(fty, llvm::Function::ExternalLinkage, "add", module.get());
  llvm::BasicBlock *entry = llvm::BasicBlock::Create(ctx, "entry", fn);
  builder.SetInsertPoint(entry);

  // 4. 绑定并命名参数
  auto arg_iter = fn->arg_begin();
  llvm::Value *x = arg_iter++;
  x->setName("x");
  llvm::Value *y = arg_iter++;
  y->setName("y");

  // 5. 创建加法与返回指令
  llvm::Value *sum = builder.CreateAdd(x, y, "sum");
  builder.CreateRet(sum);

  // 6. 打印 IR
  module->print(llvm::outs(), nullptr);
  return 0;
}
```

## MoonLLVM 目前已经做到什么？

当前，MoonLLVM 已经具备以下能力：

- 支持 LLVM IR 的构建；
- 在此基础上，完成了初步的后端代码生成，可以**独立生成 RISC-V 汇编代码，和 AArch64 汇编代码**。

基于 MoonLLVM，我们实现了一个 MiniMoonBit 编译器，除基础特性外，还支持模式匹配、高阶函数等特性，完成MiniMoonBit编译器后，我们还用 MiniMoonBit 跑了一个光线追踪程序，效果可以在 B 站视频中看到：

[B 站视频：MiniMoonBit + MoonLLVM 光线追踪示例](https://www.bilibili.com/video/BV1kSS4BqETn)

---

## 性能测试：与 tcc / clang 的对比

为了测试 MoonLLVM 的实际表现，我们设计了 5 个小例子：

- `ack.mbt`：Ackermann 递归函数；
- `fib.mbt`：递归 Fibonacci；
- `eigen.mbt`：矩阵求特征值；
- `svd.mbt`：矩阵 SVD 分解；
- `queen.mbt`：八皇后问题。

测试方式如下：

- **MiniMoonBit（基于 MoonLLVM）**

  - MiniMoonBit 生成 AArch64 汇编；
  - 使用 `clang -O0` 编译汇编文件和 `runtime.c`，得到可执行程序。
- **tcc**

  - 使用 MoonBit 将对应 `.mbt` 文件通过 `--target native` 转为 C 程序；
  - 使用 tcc 将 C 文件与 MoonBit 标准 `runtime.c` 编译为可执行程序。
- **clang -O0**

  - 同样先转为 C，再用 `clang -O0` 编译。
- **clang -O1**

  - 同样先转为 C，再用 `clang -O1` 编译。

然后分别运行所得可执行程序，记录时间（单位：秒）：

| 测试用例 | MiniMoonBit | tcc    | clang -O0 | clang -O1 |
| ---------- | ------------- | -------- | ----------- | ----------- |
| ack      | 0.011       | 0.015  | 0.013     | 0.010     |
| fib      | 0.065       | 0.169  | 0.115     | 0.036     |
| eigen    | 0.426       | 1.858  | 1.763     | 0.256     |
| svd      | 0.625       | 3.251  | 2.893     | 0.455     |
| queen    | 3.269       | 15.019 | 13.075    | 3.201     |

配图如下：

![MoonLLVM MiniMoonBit 性能对比](https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/pics-for-documents/Performace-MiniMoonbit-MoonLLVM.png)

### 性能结果分析

整体来看：

- 基于 MoonLLVM 的 MiniMoonBit 性能**明显优于 tcc 和 clang -O0**；
- 与 `clang -O1` 相比，MiniMoonBit 仍有差距，但在同数量级之内。

主要原因在于：

- **MoonLLVM 当前后端已经做了寄存器分配**（采用图着色寄存器分配）；
- 而 tcc 和 `clang -O0` **不做寄存器分配**，因此在部分算例中会出现明显性能劣势；
- `clang -O1` 在寄存器分配之外，还开启了更多优化 Pass，因此通常会比当前版本的 MiniMoonBit 更快。

换句话说，在仍然相对「年轻」的优化管线下，MoonLLVM 已经能在不少场景中跑到接近 `clang -O1` 的表现；后续优化空间依然很大。

---

## 展望：MoonLLVM 接下来要做什么？

未来一段时间内，MoonLLVM 会重点在以下方向持续演进：

1. **兑现互操作承诺：** 持续完善与真 LLVM 的互操作能力；保证「随时可以退出到真 LLVM」这条路径长期有效。
2. **扩展指令与类型系统：** 丰富 IR 指令和类型支持；增强优化能力，引入更多调试与诊断信息。
3. **增加更多体系结构后端：** 补充 `x86_64` 等后端；在更多架构上验证当前抽象方案的通用性与简洁性。
4. **打造完整工具链闭环：** 开发配套汇编器（MoonAs）与链接器（MoonLD）；构建由 MoonLLVM 驱动的、可独立运作的工具链闭环。

我们希望，MoonLLVM 既能成为教学与研究的友好平台，也能在特定场景下，成为真正落地可用的轻量级编译后端。

---

如果你对 LLVM 生态已经很熟悉，希望有一个更轻量、可实验、可与真 LLVM 平滑切换的伴侣；或者你只是想用一种更现代、更干净的方式来认识「编译器后端」这个世界，欢迎试试 MoonLLVM：

1. MoonLLVM链接：https://github.com/moonbitlang/MoonLLVM
2. MiniMoonBit编译器：https://github.com/moonbitlang/MiniMoonBit2025
3. B站视频，用MoonBit写个MiniMoonBit跑了一个光线追踪: https://www.bilibili.com/video/BV1kSS4BqETn
