---
title: LLVM Pass 剖析 - SROA
description: 剖析一下LLVM中SROA Pass的实现
image: https://blogimgs-1309485105.cos.ap-nanjing.myqcloud.com/Cover/Javascript/9.jpg
category: 技术原理
published: 2023-09-21
tags: [LLVM, Source Code Analysis, Compiler]
---

-----------

## SROA

SROA的全称叫做Scalar Replacement of Aggregates。这个优化手段按照字面意义上来说，其实是针对聚合类型的，对于C系列的语言来说，就是`struct`，通过SROA来将单个标量的空间分配和store来直接用标量来替代。

例如，对于非常简单的一行代码`int a = 1`，仅仅做语法分析而不考虑优化的话，这一个变量会在栈上占据4字节的空间，对于LLVM来说，这样的代码会有一个alloc指令和store指令。但是实际的运行过程中，我们可能会希望这个a变量一直存放在一个寄存器里面，也就是说，我们可能会希望优化alloc和store指令，而直接使用值。这就是SROA。

本篇博客来分析一下LLVM里面SROA的实现思路，在大致了解实现思路后，博客后面给出了一个功能不全但是简单易于理解的SROA实现。

## opt -debug

如果我们希望看一个现实的Pass到底是怎么工作的，我们可以使用`debug`功能。

如果需要全部的调试信息，可以直接在构建的时候就设置成`Debug`模式，然后使用`opt -debug`选项，来展示`debug`信息。

`opt -debug`会有效化`LLVM_DEBUG`宏，从而能够打印调试信息。例如，我们看一个简单的SROA优化，以下是例子：

```llvm
define i32 @foo(i32 %0, i32 %1) {
entry:
  %2 = alloca i32, align 4
  %3 = alloca i32, align 4
  store i32 %0, ptr %2, align 4
  store i32 %1, ptr %3, align 4
  %4 = load i32, ptr %2, align 4
  %5 = load i32, ptr %3, align 4
  %6 = mul i32 %4, %4
  %7 = mul i32 %5, %5
  %8 = add i32 %6, %7
  ret i32 %8
}
```

这一串代码很明显SROA优化会有作用，但是我们可能非常想知道具体是怎么转变的。我们没办法把SROA优化的整个Pass一口气全部理解，但是对于上面的简单例子，我们可能希望知道它到底是怎样进行转换的。这个时候，我们可以用`opt -debug`来辅助我们理解，使用下面的命令：

```shell
$ opt -S --passes=sroa demo.ll -debug &> logs
```

一般而言这些信息都是打印到`err`流里面的，所以这里使用`&> logs`打印到`logs`里面。

看一下logs里面的内容：

```plaintext
Args: opt -S --passes=sroa demo.ll -debug 
SROA function: foo
SROA alloca:   %3 = alloca i32, align 4
  Rewriting FCA loads and stores...
Slices of alloca:   %3 = alloca i32, align 4
  [0,4) slice #0 (splittable)
    used by:   store i32 %1, ptr %3, align 4
  [0,4) slice #1 (splittable)
    used by:   %5 = load i32, ptr %3, align 4
Pre-splitting loads and stores
  Searching for candidate loads and stores
Rewriting alloca partition [0,4) to:   %3 = alloca i32, align 4
  rewriting [0,4) slice #0 (splittable)
   Begin:(0, 4) NewBegin:(0, 4) NewAllocaBegin:(0, 4)
    original:   store i32 %1, ptr %3, align 4
          to:   store i32 %1, ptr %3, align 4
  rewriting [0,4) slice #1 (splittable)
   Begin:(0, 4) NewBegin:(0, 4) NewAllocaBegin:(0, 4)
    original:   %5 = load i32, ptr %3, align 4
          to:   %.0.load = load i32, ptr %3, align 4
  Speculating PHIs
  Rewriting Selects
Deleting dead instruction:   %5 = load i32, ptr %3, align 4
Deleting dead instruction:   store i32 %1, ptr %3, align 4
SROA alloca:   %2 = alloca i32, align 4
  Rewriting FCA loads and stores...
Slices of alloca:   %2 = alloca i32, align 4
  [0,4) slice #0 (splittable)
    used by:   store i32 %0, ptr %2, align 4
  [0,4) slice #1 (splittable)
    used by:   %4 = load i32, ptr %2, align 4
Pre-splitting loads and stores
  Searching for candidate loads and stores
Rewriting alloca partition [0,4) to:   %2 = alloca i32, align 4
  rewriting [0,4) slice #0 (splittable)
   Begin:(0, 4) NewBegin:(0, 4) NewAllocaBegin:(0, 4)
    original:   store i32 %0, ptr %2, align 4
          to:   store i32 %0, ptr %2, align 4
  rewriting [0,4) slice #1 (splittable)
   Begin:(0, 4) NewBegin:(0, 4) NewAllocaBegin:(0, 4)
    original:   %4 = load i32, ptr %2, align 4
          to:   %.0.load1 = load i32, ptr %2, align 4
  Speculating PHIs
  Rewriting Selects
Deleting dead instruction:   %4 = load i32, ptr %2, align 4
Deleting dead instruction:   store i32 %0, ptr %2, align 4
Promoting allocas with mem2reg...
; ModuleID = 'demo.ll'
source_filename = "demo.ll"

define i32 @foo(i32 %0, i32 %1) {
entry:
  %2 = mul i32 %0, %0
  %3 = mul i32 %1, %1
  %4 = add i32 %2, %3
  ret i32 %4
}
```

如果仅仅是看代码，可能会很容易把目光放在一个`rewrite`函数上，但是实际上，我们会发现这里的`rewrite`并没有起到什么作用，基本上是原样复制一份了。SROA本身的想法是对Aggregate类型做拆分，但是我们这里没有涉及到Aggregate类型，所以`rewrite`并没有作用。

真正起到作用的，应该是其中的一句话`Promoting allocas with mem2reg...`。

## PromoteMem2Reg

在这个线索之下，再去看源代码，就会发现下面的函数：

```cpp
/// Promote the allocas, using the best available technique.
///
/// This attempts to promote whatever allocas have been identified as viable in
/// the PromotableAllocas list. If that list is empty, there is nothing to do.
/// This function returns whether any promotion occurred.
bool SROAPass::promoteAllocas(Function &F) {
  if (PromotableAllocas.empty())
    return false;

  NumPromoted += PromotableAllocas.size();

  LLVM_DEBUG(dbgs() << "Promoting allocas with mem2reg...\n");
  PromoteMemToReg(PromotableAllocas, DTU->getDomTree(), AC);
  PromotableAllocas.clear();
  return true;
}

```

所以，造成前面LLVM代码转变的主逻辑实际上在这个`PromoteMemToReg`上。在`llvm/Transform/Utils/PromoteMemToRge.h`找到了下面的内容：

```cpp
/// Return true if this alloca is legal for promotion.
///
/// This is true if there are only loads, stores, and lifetime markers
/// (transitively) using this alloca. This also enforces that there is only
/// ever one layer of bitcasts or GEPs between the alloca and the lifetime
/// markers.
bool isAllocaPromotable(const AllocaInst *AI);

/// Promote the specified list of alloca instructions into scalar
/// registers, inserting PHI nodes as appropriate.
///
/// This function makes use of DominanceFrontier information.  This function
/// does not modify the CFG of the function at all.  All allocas must be from
/// the same function.
///
void PromoteMemToReg(ArrayRef<AllocaInst *> Allocas, DominatorTree &DT,
                     AssumptionCache *AC = nullptr);
```

而一旦知道了这个事实，我们就可以在我们的code中，插入类似于如下的语句：

```cpp
  FunctionAnalysisManager FAM;
  DominatorTreeAnalysis DTA;
  DominatorTree DT = DTA.run(*foo, FAM);
  PromoteMemToReg(allocas, DT);
```

然后我们就会发现我们的代码就被优化掉成我们希望的那个样子了，这里我们把整体的代码贴在下面：

```cpp
#include <llvm/Analysis/DomTreeUpdater.h>
#include <llvm/IR/Dominators.h>
#include <llvm/IR/IRBuilder.h>
#include <llvm/IR/Instructions.h>
#include <llvm/IR/LLVMContext.h>
#include <llvm/IR/LegacyPassManager.h>
#include <llvm/IR/Module.h>
#include <llvm/IR/Verifier.h>
#include <llvm/Pass.h>
#include <llvm/Support/raw_ostream.h>
#include <llvm/Transforms/Scalar.h>
#include <llvm/Transforms/Utils/PromoteMemToReg.h>
#include <memory>

using namespace llvm;

int main() {
  LLVMContext context;
  auto demo = std::make_unique<Module>("demo", context);
  IRBuilder<> builder(context);
  Type *int32ty = builder.getInt32Ty();
  FunctionType *ft = FunctionType::get(int32ty, {int32ty, int32ty}, false);
  // int foo(int a, int b) {int x = a; int y = b; int r = x*x+y*y; return r;}
  Function *foo =
      Function::Create(ft, Function::ExternalLinkage, "foo", demo.get());

  BasicBlock *entry = BasicBlock::Create(context, "entry", foo);
  builder.SetInsertPoint(entry);
  Value *a = foo->arg_begin();
  Value *b = foo->arg_begin() + 1;
  Value *xp = builder.CreateAlloca(int32ty);
  Value *yp = builder.CreateAlloca(int32ty);
  builder.CreateStore(a, xp);
  builder.CreateStore(b, yp);
  Value *xv = builder.CreateLoad(int32ty, xp);
  Value *yv = builder.CreateLoad(int32ty, yp);
  Value *x2 = builder.CreateMul(xv, xv);
  Value *y2 = builder.CreateMul(yv, yv);
  Value *r = builder.CreateAdd(x2, y2);
  builder.CreateRet(r);

  // auto FPM = std::make_unique<legacy::FunctionPassManager>(demo.get());
  // FPM->add(createSROAPass());
  // FPM->doInitialization();
  // FPM->run(*foo);

  outs() << "==========Before promotion===============\n";
  demo->print(outs(), nullptr);

  AllocaInst *in1 = dyn_cast<AllocaInst>(xp);
  AllocaInst *in2 = dyn_cast<AllocaInst>(yp);
  SmallVector<AllocaInst *, 2> allocas;
  allocas.push_back(in1);
  allocas.push_back(in2);
  FunctionAnalysisManager FAM;
  DominatorTreeAnalysis DTA;
  DominatorTree DT = DTA.run(*foo, FAM);
  PromoteMemToReg(allocas, DT);

  outs() << "==========After promotion===============\n";
  demo->print(outs(), nullptr);

  return 0;
}
```

打印一下看看：

```plaintext
==========Before promotion===============
; ModuleID = 'demo'
source_filename = "demo"

define i32 @foo(i32 %0, i32 %1) {
entry:
  %2 = alloca i32, align 4
  %3 = alloca i32, align 4
  store i32 %0, ptr %2, align 4
  store i32 %1, ptr %3, align 4
  %4 = load i32, ptr %2, align 4
  %5 = load i32, ptr %3, align 4
  %6 = mul i32 %4, %4
  %7 = mul i32 %5, %5
  %8 = add i32 %6, %7
  ret i32 %8
}
==========After promotion===============
; ModuleID = 'demo'
source_filename = "demo"

define i32 @foo(i32 %0, i32 %1) {
entry:
  %2 = mul i32 %0, %0
  %3 = mul i32 %1, %1
  %4 = add i32 %2, %3
  ret i32 %4
}
```

所以，如果想要进一步知道它的变换历程，就需要去查看这个`PromoteMemToReg`，通过源代码可以看到下面这个函数：

```cpp
void llvm::PromoteMemToReg(ArrayRef<AllocaInst *> Allocas, DominatorTree &DT,
                           AssumptionCache *AC) {
  // If there is nothing to do, bail out...
  if (Allocas.empty())
    return;

  PromoteMem2Reg(Allocas, DT, AC).run();
}
```

看来还有一个`PromoteMem2Reg`类，然后这个类有一个run函数：

```cpp
struct PromoteMem2Reg {
//...
public:
  PromoteMem2Reg(ArrayRef<AllocaInst *> Allocas, DominatorTree &DT,
                 AssumptionCache *AC)
      : Allocas(Allocas.begin(), Allocas.end()), DT(DT),
        DIB(*DT.getRoot()->getParent()->getParent(), /*AllowUnresolved*/ false),
        AC(AC), SQ(DT.getRoot()->getParent()->getParent()->getDataLayout(),
                   nullptr, &DT, AC) {}

  void run();
//...
}
```

浏览这个`run`函数，会发现它会遍历每一个`Alloca`指令，然后，跟我们的代码相关的是下面几行：

```cpp
AllocaInfo Info;
//... for-loop

// Calculate the set of read and write-locations for each alloca.  This is
// analogous to finding the 'uses' and 'definitions' of each variable.
Info.AnalyzeAlloca(AI);

// If there is only a single store to this value, replace any loads of
// it that are directly dominated by the definition with the value stored.
if (Info.DefiningBlocks.size() == 1) {
    if (rewriteSingleStoreAlloca(AI, Info, LBI, SQ.DL, DT, AC)) {
        // The alloca has been processed, move on.
        RemoveFromAllocasList(AllocaNum);
        ++NumSingleStore;
        continue;
    }
}
```

也就是先分析了一下这个`Alloca`，如果发现只有一个`store`，也就是这里的`Info.DefiningBlocks.size()==1`，就会调用`rewriteSingleStoreAlloca`。然后，在这个函数里面找到了以下的几行：

```cpp
StoreInst *OnlyStore = Info.OnlyStore;

  for (User *U : make_early_inc_range(AI->users())) {
    Instruction *UserInst = cast<Instruction>(U);
    if (UserInst == OnlyStore)
      continue;
    LoadInst *LI = cast<LoadInst>(UserInst);

    // Otherwise, we *can* safely rewrite this load.
    Value *ReplVal = OnlyStore->getOperand(0);
    // If the replacement value is the load, this must occur in unreachable
    // code.
    if (ReplVal == LI)
      ReplVal = PoisonValue::get(LI->getType());

    convertMetadataToAssumes(LI, ReplVal, DL, AC, &DT);
    LI->replaceAllUsesWith(ReplVal);
    LI->eraseFromParent();
    LBI.deleteValue(LI);
  }

```

那么也就是说，如果只有一个store的话，这个函数会尝试把每一个Load直接替换成原先store进去的那个值，然后再去掉这条指令。

然后，观察一下`AllocaInfo::AnalyzeAlloca`这个函数：

```cpp
  void clear() {
    DefiningBlocks.clear();
    UsingBlocks.clear();
    OnlyStore = nullptr;
    OnlyBlock = nullptr;
    OnlyUsedInOneBlock = true;
    DbgUsers.clear();
    AssignmentTracking.clear();
  }
  /// Scan the uses of the specified alloca, filling in the AllocaInfo used
  /// by the rest of the pass to reason about the uses of this alloca.
  void AnalyzeAlloca(AllocaInst *AI) {
    clear();

    // As we scan the uses of the alloca instruction, keep track of stores,
    // and decide whether all of the loads and stores to the alloca are within
    // the same basic block.
    for (User *U : AI->users()) {
      Instruction *User = cast<Instruction>(U);

      if (StoreInst *SI = dyn_cast<StoreInst>(User)) {
        // Remember the basic blocks which define new values for the alloca
        DefiningBlocks.push_back(SI->getParent());
        OnlyStore = SI;
      } else {
        LoadInst *LI = cast<LoadInst>(User);
        // Otherwise it must be a load instruction, keep track of variable
        // reads.
        UsingBlocks.push_back(LI->getParent());
      }

      if (OnlyUsedInOneBlock) {
        if (!OnlyBlock)
          OnlyBlock = User->getParent();
        else if (OnlyBlock != User->getParent())
          OnlyUsedInOneBlock = false;
      }
    }
    DbgUserVec AllDbgUsers;
    findDbgUsers(AllDbgUsers, AI);
    std::copy_if(AllDbgUsers.begin(), AllDbgUsers.end(),
                 std::back_inserter(DbgUsers), [](DbgVariableIntrinsic *DII) {
                   return !isa<DbgAssignIntrinsic>(DII);
                 });
    AssignmentTracking.init(AI);
  }
```

其实在这里，对我们最有用的，还是在于得到这个OnlyStore的过程。而就整个函数来说，它要得到的是这个哪些块`store`，那些块`load`，是否仅仅被使用一次等等。

## MySROA

一旦得知了它的思路，我们就有想法来得到我们自己的SROA优化，当然，是最简单的那种：

```cpp
class MySROA {
private:
  SmallVector<AllocaInst *, 8> *Allocas;
  Function *F;

public:
  MySROA(Function *F) : F(F) {}

  void run() {
    MyVisitor visitor(Allocas);
    visitor.visit(F);

    if (Allocas->empty())
      return;

    MyPromote promote(Allocas, F);
    promote.run();
  }
};
```

首先有一个`MySROA`类来托管`Function`，然后通过`Visitor`来遍历所有的`alloca`指令。接着使用`MyPromote`来进行`store`和`load`的替换：

```cpp
// Only visit alloca
class MyVisitor : public InstVisitor<MyVisitor, bool> {
public:
  SmallVector<AllocaInst *, 8> *Allocas;
  MyVisitor(SmallVector<AllocaInst *, 8> *Allocas) : Allocas(Allocas) {}

  // do nothing if it's not alloca
  bool visitInstruction(Instruction &I) { return false; }

  bool visitAllocaInst(AllocaInst &I) {
    Allocas->push_back(&I);
    return true;
  }
};
```

`MyVisitor`比较简单，主要就是收集所有的`AllocaInst`到一个`SmallVector`里面。

然后，进入到`MyPromote`里面：

```cpp
class MyPromote {
private:
  Function *F;
  SmallVector<AllocaInst *, 8> &Allocas;

  class AllocaInfo {
  public:
    StoreInst *onlyStore;

    void AnalysisAlloc(AllocaInst *AI) {
      onlyStore = nullptr;
      for (User *U : AI->users()) {
        Instruction *User = cast<Instruction>(U);
        if (StoreInst *SI = dyn_cast<StoreInst>(User)) {
          if (onlyStore == nullptr) {
            onlyStore = SI;
          } else {
            onlyStore = nullptr;
            return;
          }
        }
      }
    }
  };

  AllocaInfo Info;

public:
  MyPromote(SmallVector<AllocaInst *, 8> *Allocas, Function *F)
      : Allocas(*Allocas), F(F) {}

  void rewriteSingleStoreAlloca(AllocaInst *AI, StoreInst *OnlyStore) {
    for (User *U : make_early_inc_range(AI->users())) {
      Instruction *User = cast<Instruction>(U);
      if (User == OnlyStore)
        continue;

      LoadInst *LI = dyn_cast<LoadInst>(User);
      Value *ReplVal = OnlyStore->getOperand(0);

      // TODO:if (ReplVal == LI)
      LI->replaceAllUsesWith(ReplVal);
      LI->eraseFromParent();
    }
  }

  void removeDeadStore(StoreInst *SI) {
    Value *Val = SI->getOperand(1);
    bool isDead = true;
    for (User *U : Val->users()) {
      if (U != SI) {
        isDead = false;
        break;
      }
    }
    if (isDead) {
      SI->eraseFromParent();
    }
  }

  void run() {

    SmallVector<unsigned, 8> RemoveList;
    for (unsigned i = 0; i < Allocas.size(); i++) {
      AllocaInst *AI = Allocas[i];
      Info.AnalysisAlloc(AI);

      if (Info.onlyStore == nullptr)
        continue;

      // if onlystore, try to replace all load with the value
      rewriteSingleStoreAlloca(AI, Info.onlyStore);
      // remove dead store
      removeDeadStore(Info.onlyStore);
      // Mark the alloca to be removed
      RemoveList.push_back(i);
    }

    for (unsigned j : RemoveList) {
      if (Allocas[j]->use_empty())
        Allocas[j]->eraseFromParent();
    }
  }
};
```

主要是`run`函数，遍历每一个`Alloca`指令，分析它是否只有一次`store`，如果确实只有一次`store`，那么通过`rewriteSingleStoreAlloca`，把每一个`load`都给替换成这个原先store的这个值。然后尝试进行`removeStore`，最后再尝试去掉整个`Alloca`指令。

## Full Code

```cpp
#include <llvm/IR/IRBuilder.h>
#include <llvm/IR/InstVisitor.h>
#include <llvm/IR/Instructions.h>
#include <llvm/IR/LLVMContext.h>
#include <llvm/IR/Module.h>
#include <memory>

using namespace llvm;

// Only visit alloca
class MyVisitor : public InstVisitor<MyVisitor, bool> {
public:
  SmallVector<AllocaInst *, 8> *Allocas;
  MyVisitor(SmallVector<AllocaInst *, 8> *Allocas) : Allocas(Allocas) {}

  // do nothing if it's not alloca
  bool visitInstruction(Instruction &I) { return false; }

  bool visitAllocaInst(AllocaInst &I) {
    Allocas->push_back(&I);
    return true;
  }
};

class MyPromote {
private:
  Function *F;
  SmallVector<AllocaInst *, 8> &Allocas;

  class AllocaInfo {
  public:
    StoreInst *onlyStore;

    void AnalysisAlloc(AllocaInst *AI) {
      onlyStore = nullptr;
      for (User *U : AI->users()) {
        Instruction *User = cast<Instruction>(U);
        if (StoreInst *SI = dyn_cast<StoreInst>(User)) {
          if (onlyStore == nullptr) {
            onlyStore = SI;
          } else {
            onlyStore = nullptr;
            return;
          }
        }
      }
    }
  };

  AllocaInfo Info;

public:
  MyPromote(SmallVector<AllocaInst *, 8> *Allocas, Function *F)
      : Allocas(*Allocas), F(F) {}

  void rewriteSingleStoreAlloca(AllocaInst *AI, StoreInst *OnlyStore) {
    for (User *U : make_early_inc_range(AI->users())) {
      Instruction *User = cast<Instruction>(U);
      if (User == OnlyStore)
        continue;

      LoadInst *LI = dyn_cast<LoadInst>(User);
      Value *ReplVal = OnlyStore->getOperand(0);

      // TODO:if (ReplVal == LI)
      LI->replaceAllUsesWith(ReplVal);
      LI->eraseFromParent();
    }
  }

  void removeDeadStore(StoreInst *SI) {
    Value *Val = SI->getOperand(1);
    bool isDead = true;
    for (User *U : Val->users()) {
      if (U != SI) {
        isDead = false;
        break;
      }
    }
    if (isDead) {
      SI->eraseFromParent();
    }
  }

  void run() {

    SmallVector<unsigned, 8> RemoveList;
    for (unsigned i = 0; i < Allocas.size(); i++) {
      AllocaInst *AI = Allocas[i];
      Info.AnalysisAlloc(AI);

      if (Info.onlyStore == nullptr)
        continue;

      // if onlystore, try to replace all load with the value
      rewriteSingleStoreAlloca(AI, Info.onlyStore);
      // remove dead store
      removeDeadStore(Info.onlyStore);
      // Mark the alloca to be removed
      RemoveList.push_back(i);
    }

    for (unsigned j : RemoveList) {
      if (Allocas[j]->use_empty())
        Allocas[j]->eraseFromParent();
    }
  }
};

class MySROA {
private:
  SmallVector<AllocaInst *, 8> *Allocas;
  Function *F;

public:
  MySROA(Function *F) : F(F) {}

  void run() {
    MyVisitor visitor(Allocas);
    visitor.visit(F);

    if (Allocas->empty())
      return;

    MyPromote promote(Allocas, F);
    promote.run();
  }
};

int main() {
  LLVMContext context;
  auto demo = std::make_unique<Module>("demo", context);
  IRBuilder<> builder(context);
  Type *int32ty = builder.getInt32Ty();
  FunctionType *ft = FunctionType::get(int32ty, {int32ty, int32ty}, false);
  // int foo(int a, int b) {int x = a; int y = b; int r = x*x+y*y; return r;}
  Function *foo =
      Function::Create(ft, Function::ExternalLinkage, "foo", demo.get());

  BasicBlock *entry = BasicBlock::Create(context, "entry", foo);
  builder.SetInsertPoint(entry);
  Value *a = foo->arg_begin();
  Value *b = foo->arg_begin() + 1;
  Value *xp = builder.CreateAlloca(int32ty);
  Value *yp = builder.CreateAlloca(int32ty);
  builder.CreateStore(a, xp);
  builder.CreateStore(b, yp);
  Value *xv = builder.CreateLoad(int32ty, xp);
  Value *yv = builder.CreateLoad(int32ty, yp);
  Value *x2 = builder.CreateMul(xv, xv);
  Value *y2 = builder.CreateMul(yv, yv);
  Value *r = builder.CreateAdd(x2, y2);
  builder.CreateRet(r);

  outs() << "===========Before My SROA:==============\n";
  demo->print(outs(), nullptr);

  MySROA sroa(foo);
  sroa.run();

  outs() << "===========After My SROA:==============\n";
  demo->print(outs(), nullptr);
  return 0;
}
```
