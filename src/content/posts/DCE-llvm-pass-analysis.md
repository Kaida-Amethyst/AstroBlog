---
title: LLVM Pass 剖析 - DCE
description: 剖析一下LLVM中DCE Pass的实现
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Javascript/8.jpg
category: 技术原理
published: 2023-08-26
tags: [LLVM, Source Code Analysis, Compiler]
---

------------------------

## Dead Code Elimination

Dead Code Elimination，死代码删除（DCE），作用是将代码中实际上不起作用的代码进行消去，从而提升性能，例如下面的代码：

```cpp
int foo(int a) {
    int n = a + 1;  // Dead Code
	int m = a >> 2;
	return m;
}
```

上面的代码中就出现了一个无效的代码，`int n = a + 1;`，这一行代码在整个函数里面实际上没有什么意义，因此被称为死代码，Dead Code，优化思路就是将这一行代码去除。

## Idea

基本的思路是通过du链和ud链来进行判断。也就是说，对于一条指令，如何知道它是死代码呢？如果这一条指令的结果没有被使用，那么这一条指令就是无效的指令。将其删除出去。

那么，对于这条指令而言，它可能会有多个操作数，每个操作数又会来源于某条指令，当这条指令被删除之后，它得操作数所来源的指令可能也会因为这条指令的删除，而使得指令也变成死代码。例如下面的代码：

```cpp
int foo(int a) {
    int b = a >> 1;
	  int n = b * a;
	  int m = a >> 2;
  	return m;
}
```

对于上面的代码而言，n没有被使用，因此会被删除，而一旦`int n = b * a;`被删除，`int b = a >> 1;`也会变成死代码，因此也会被删除。

这个过程需要注意的，就是在ud链和du链构建的时候，有一些指令可能是符合“没有被使用”的，但是却不应该被删除，例如，上面代码中的最后一行`return m`，它在一些du链和ud链中是没有后继使用者的，但是却应该被保留。

## LLVM Pass Code

弄明白思路之后，接下来需要做的就是实现。我们遍历函数中的每一条指令，如果发现某一条指令是出现操作结果没有被使用的情况，就删除这条指令，并且记录这条指令的操作数，将它们加入到WorkList中，查看它们是否也是dead的，如果是，就继续执行清除。

### elimiateDeadCode

```cpp
static bool eliminateDeadCode(Function &F, TargetLibraryInfo *TLI) {
  bool MadeChange = false;
  SmallSetVector<Instruction *, 16> WorkList;
  // Iterate over the original function, only adding insts to the worklist
  // if they actually need to be revisited. This avoids having to pre-init
  // the worklist with the entire function's worth of instructions.
  for (Instruction &I : llvm::make_early_inc_range(instructions(F))) {
    // We're visiting this instruction now, so make sure it's not in the
    // worklist from an earlier visit.
    if (!WorkList.count(&I))
      MadeChange |= DCEInstruction(&I, WorkList, TLI);
  }

  while (!WorkList.empty()) {
    Instruction *I = WorkList.pop_back_val();
    MadeChange |= DCEInstruction(I, WorkList, TLI);
  }
  return MadeChange;
}
```

首先对函数内的指令进行遍历，首先将一部分指令进行消除，使用WorkList保存这些指令的操作数。

然后再针对WorkList中的指令，查看它们是否是可以被删除的，一直循环到WorkList为空，程序结束。

### DCEInstruction

```cpp
static bool DCEInstruction(Instruction *I,
                           SmallSetVector<Instruction *, 16> &WorkList,
                           const TargetLibraryInfo *TLI) {
  if (isInstructionTriviallyDead(I, TLI)) {
    if (!DebugCounter::shouldExecute(DCECounter))
      return false;

    salvageDebugInfo(*I);
    salvageKnowledge(I);

    // Null out all of the instruction's operands to see if any operand becomes
    // dead as we go.
    for (unsigned i = 0, e = I->getNumOperands(); i != e; ++i) {
      Value *OpV = I->getOperand(i);
      I->setOperand(i, nullptr);

      if (!OpV->use_empty() || I == OpV)
        continue;

      // If the operand is an instruction that became dead as we nulled out the
      // operand, and if it is 'trivially' dead, delete it in a future loop
      // iteration.
      if (Instruction *OpI = dyn_cast<Instruction>(OpV))
        if (isInstructionTriviallyDead(OpI, TLI))
          WorkList.insert(OpI);
    }

    I->eraseFromParent();
    ++DCEEliminated;
    return true;
  }
  return false;
}
```

在DCEInstruction函数里面，首先会查看这个函数是否是`trivialDead`，通俗来说就是use为空的情况，如果是的话，那就要进行消除。在消除前，会首先遍历这条指令的所有操作数，如果操作数存在use不为空的情况，那么这个操作数至少是暂时不能判定为死代码的，就需要跳过循环。如果这个操作数的use也为空，那么还需要`isInstructionTriviallyDead`函数的再次判定，如果确实为空，那么就需要把这个操作数的指令给加入到`WorkList`当中，在`eliminateDeadCode`里面进行处理。

接下来需要讨论的就是`isInstructionTriviallyDead`这个函数。

### isInstructionTriviallyDead

对于一个指令是否是Trivially Dead，首先要判断的是这个指令的use是否为空，如果use不为空，那么它一定不是Trivially Dead。但如果是空的，此时需要进一步判断这个指令的类型，有些情况下，use为空，但本身可能并非是死代码，例如一个函数的return语句，它的use为空，但是却并非是Trivially Dead的。

```cpp
bool llvm::isInstructionTriviallyDead(Instruction *I,
                                      const TargetLibraryInfo *TLI) {
  if (!I->use_empty())
    return false;
  return wouldInstructionBeTriviallyDead(I, TLI);
}
```

这里之后调用了`wouldInstructionBeTriviallyDead`，看一下这个函数的逻辑：

```cpp
bool llvm::wouldInstructionBeTriviallyDeadOnUnusedPaths(
    Instruction *I, const TargetLibraryInfo *TLI) {
  // Instructions that are "markers" and have implied meaning on code around
  // them (without explicit uses), are not dead on unused paths.
  if (IntrinsicInst *II = dyn_cast<IntrinsicInst>(I))
    if (II->getIntrinsicID() == Intrinsic::stacksave ||
        II->getIntrinsicID() == Intrinsic::launder_invariant_group ||
        II->isLifetimeStartOrEnd())
      return false;
  return wouldInstructionBeTriviallyDead(I, TLI);
}
```

在这个函数里面，会把指令尝试转换为IntrinsicInst，如果成功了，再次判断这种IntrinsiciInst是否特定的指令，如果是，那么必须保留，所以会直接返回false。

如果指令不能被转换为IntrinsicInst，那么还需要进一步查看。

### wouldInstructionBeTriviallyDead

有一些指令满足`use_empty`，但是它们不能被删除，在`wouldInstructionBeTriviallyDead`中，会对这些情况做一个汇总：

首先把整体的代码贴下：

```cpp
bool llvm::wouldInstructionBeTriviallyDead(const Instruction *I,
                                           const TargetLibraryInfo *TLI) {
  if (I->isTerminator())
    return false;

  // We don't want the landingpad-like instructions removed by anything this
  // general.
  if (I->isEHPad())
    return false;

  // We don't want debug info removed by anything this general.
  if (isa<DbgVariableIntrinsic>(I))
    return false;

  if (const DbgLabelInst *DLI = dyn_cast<DbgLabelInst>(I)) {
    if (DLI->getLabel())
      return false;
    return true;
  }

  if (auto *CB = dyn_cast<CallBase>(I))
    if (isRemovableAlloc(CB, TLI))
      return true;

  if (!I->willReturn()) {
    auto *II = dyn_cast<IntrinsicInst>(I);
    if (!II)
      return false;

    switch (II->getIntrinsicID()) {
    case Intrinsic::experimental_guard: {
      // Guards on true are operationally no-ops.  In the future we can
      // consider more sophisticated tradeoffs for guards considering potential
      // for check widening, but for now we keep things simple.
      auto *Cond = dyn_cast<ConstantInt>(II->getArgOperand(0));
      return Cond && Cond->isOne();
    }
    // TODO: These intrinsics are not safe to remove, because this may remove
    // a well-defined trap.
    case Intrinsic::wasm_trunc_signed:
    case Intrinsic::wasm_trunc_unsigned:
    case Intrinsic::ptrauth_auth:
    case Intrinsic::ptrauth_resign:
      return true;
    default:
      return false;
    }
  }

  if (!I->mayHaveSideEffects())
    return true;

  // Special case intrinsics that "may have side effects" but can be deleted
  // when dead.
  if (const IntrinsicInst *II = dyn_cast<IntrinsicInst>(I)) {
    // Safe to delete llvm.stacksave and launder.invariant.group if dead.
    if (II->getIntrinsicID() == Intrinsic::stacksave ||
        II->getIntrinsicID() == Intrinsic::launder_invariant_group)
      return true;

    // Intrinsics declare sideeffects to prevent them from moving, but they are
    // nops without users.
    if (II->getIntrinsicID() == Intrinsic::allow_runtime_check ||
        II->getIntrinsicID() == Intrinsic::allow_ubsan_check)
      return true;

    if (II->isLifetimeStartOrEnd()) {
      auto *Arg = II->getArgOperand(1);
      // Lifetime intrinsics are dead when their right-hand is undef.
      if (isa<UndefValue>(Arg))
        return true;
      // If the right-hand is an alloc, global, or argument and the only uses
      // are lifetime intrinsics then the intrinsics are dead.
      if (isa<AllocaInst>(Arg) || isa<GlobalValue>(Arg) || isa<Argument>(Arg))
        return llvm::all_of(Arg->uses(), [](Use &Use) {
          if (IntrinsicInst *IntrinsicUse =
                  dyn_cast<IntrinsicInst>(Use.getUser()))
            return IntrinsicUse->isLifetimeStartOrEnd();
          return false;
        });
      return false;
    }

    // Assumptions are dead if their condition is trivially true.
    if (II->getIntrinsicID() == Intrinsic::assume &&
        isAssumeWithEmptyBundle(cast<AssumeInst>(*II))) {
      if (ConstantInt *Cond = dyn_cast<ConstantInt>(II->getArgOperand(0)))
        return !Cond->isZero();

      return false;
    }

    if (auto *FPI = dyn_cast<ConstrainedFPIntrinsic>(I)) {
      std::optional<fp::ExceptionBehavior> ExBehavior =
          FPI->getExceptionBehavior();
      return *ExBehavior != fp::ebStrict;
    }
  }

  if (auto *Call = dyn_cast<CallBase>(I)) {
    if (Value *FreedOp = getFreedOperand(Call, TLI))
      if (Constant *C = dyn_cast<Constant>(FreedOp))
        return C->isNullValue() || isa<UndefValue>(C);
    if (isMathLibCallNoop(Call, TLI))
      return true;
  }

  // Non-volatile atomic loads from constants can be removed.
  if (auto *LI = dyn_cast<LoadInst>(I))
    if (auto *GV = dyn_cast<GlobalVariable>(
            LI->getPointerOperand()->stripPointerCasts()))
      if (!LI->isVolatile() && GV->isConstant())
        return true;

  return false;
}

```

来仔细阅读上面的代码，可以看到这个代码里面被分解成了多个if判断，每个判断去断定这个指令是否可以被去除。

```cpp
  if (I->isTerminator())
    return false;
```

对于return语句，或者分支语句，不应该去除。

```cpp
  if (I->isEHPad())
    return false;
```

对于异常处理块，不应该被去除。

```cpp
  if (isa<DbgVariableIntrinsic>(I))
    return false;

  if (const DbgLabelInst *DLI = dyn_cast<DbgLabelInst>(I)) {
    if (DLI->getLabel())
      return false;
    return true;
  }
```

如果指令与调试信息相关，尽管可能不参与程序的功能，但是也不应该被删除。

```cpp
  if (auto *CB = dyn_cast<CallBase>(I))
    if (isRemovableAlloc(CB, TLI))
      return true;
```

对于某些无效的内存分配，也应该被认为是可以被移除的。

```cpp
  if (!I->willReturn()) {
    auto *II = dyn_cast<IntrinsicInst>(I);
    if (!II)
      return false;

    switch (II->getIntrinsicID()) {
    case Intrinsic::experimental_guard: {
      // Guards on true are operationally no-ops.  In the future we can
      // consider more sophisticated tradeoffs for guards considering potential
      // for check widening, but for now we keep things simple.
      auto *Cond = dyn_cast<ConstantInt>(II->getArgOperand(0));
      return Cond && Cond->isOne();
    }
    // TODO: These intrinsics are not safe to remove, because this may remove
    // a well-defined trap.
    case Intrinsic::wasm_trunc_signed:
    case Intrinsic::wasm_trunc_unsigned:
    case Intrinsic::ptrauth_auth:
    case Intrinsic::ptrauth_resign:
      return true;
    default:
      return false;
    }
  }
```

对于非返回的指令，如果它属于IntrinsicInst，还需要根据具体的指令类型来决定是否执行删除操作。

```cpp
  if (!I->mayHaveSideEffects())
    return true;
```

副作用检查，如果没有副作用，应该被认为是可以删除的。

```cpp
if (const IntrinsicInst *II = dyn_cast<IntrinsicInst>(I)) {
  //...
}
```

特殊的IntrinsicInst的处理，这里需要考虑多种IntrinsicInst，不赘述了。

```cpp
  if (auto *Call = dyn_cast<CallBase>(I)) {
    if (Value *FreedOp = getFreedOperand(Call, TLI))
      if (Constant *C = dyn_cast<Constant>(FreedOp))
        return C->isNullValue() || isa<UndefValue>(C);
    if (isMathLibCallNoop(Call, TLI))
      return true;
  }
```

一些特殊的函数调用，需要查看函数调用的类型来判断是否需要执行删除。

```cpp
  // Non-volatile atomic loads from constants can be removed.
  if (auto *LI = dyn_cast<LoadInst>(I))
    if (auto *GV = dyn_cast<GlobalVariable>(
            LI->getPointerOperand()->stripPointerCasts()))
      if (!LI->isVolatile() && GV->isConstant())
        return true;
```

最后是`Volatile`或者一些加载指令的判断。
