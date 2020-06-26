# microtask 队列与 async/await 源码分析
本文先分析 microtask 队列，后分析 async/await，版本为 V8 7.7.1。
## microtask 队列

microtask 队列存在于 V8 中，从 Node 和 V8 源码来看，V8 有暴露 microtask 队列的相关方法给 Node。也就是说 Node 可以控制 V8 的 microtask 队列，比如向 microtask 队列新增一个 microtask，或者遍历 microtask 队列。
### 基础功能

当执行以下代码：
```JavaScript
Promise.resolve('abc').then(res => console.log(res))
```

V8 的 microtask 队列里会新增一个 microtask，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/execution/microtask-queue.cc#100)：

```c++
void MicrotaskQueue::EnqueueMicrotask(Microtask microtask) {
  if (size_ == capacity_) {
    // size_:microtask 队列中 microtask 的个数
    // capacity_:microtask 队列的容量
    // size_ 等于 capacity_ 说明容量不够，需要扩容
    intptr_t new_capacity = std::max(kMinimumCapacity, capacity_ << 1); // 得到新容量
    ResizeBuffer(new_capacity); // 开始扩容
  }

  DCHECK_LT(size_, capacity_);
  // ring_buffer_:存放所有的 microtask
  ring_buffer_[(start_ + size_) % capacity_] = microtask.ptr();
  ++size_; // 个数自增 1
}
```
MicrotaskQueue 类是 microtask 队列在 V8 中的抽象表示，size_ 表示 microtask 队列中 microtask 的个数，capacity_ 表示 microtask 队列的容量，如果 size_ 等于 capacity_，表示 microtask 队列容量不够，需要扩容。调用 ResizeBuffer 方法给 microtask 队列扩容。

如果 microtask 队列容量足够，则向 ring_buffer_ 中存入当前的 microtask，ring_buffer_ 是一个指针，通过 ring_buffer_ 可以找到所有的 microtask，ring_buffer_ 定义如下：

```c++
Address* ring_buffer_ = nullptr;
```

上文提到的给 microtask 队列扩容的 ResizeBuffer 方法[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/execution/microtask-queue.cc#261)：

```c++
void MicrotaskQueue::ResizeBuffer(intptr_t new_capacity) {
  DCHECK_LE(size_, new_capacity);
  Address* new_ring_buffer = new Address[new_capacity]; // 新申请一个容量更大的数组
  for (intptr_t i = 0; i < size_; ++i) { // 把 ring_buffer_ 指向的所有 microtask 全部复制到 new_ring_buffer
    new_ring_buffer[i] = ring_buffer_[(start_ + i) % capacity_];
  }

  delete[] ring_buffer_; // 释放旧的 ring_buffer_ 的内存
  ring_buffer_ = new_ring_buffer; // ring_buffer_ 开始指向新的数组
  capacity_ = new_capacity; // 容量更新为新的容量
  start_ = 0;
}
```

ResizeBuffer 方法的逻辑很简单，总结如下：

- 新申请一个容量更大的数组，new_ring_buffer 去指向它
- 把 ring_buffer_ 指向的全部 microtask，复制到 new_ring_buffer
- 释放 ring_buffer_ 指向的内存
- ring_buffer_ 指向新的数组
- 容量更新为新的容量

> microtask 队列存在于 V8 中，V8 有暴露 microtask 队列的方法给 Node
>
> microtask 队列的底层数据结构为数组，或者动态数组


### 奇技淫巧（建议跳过不看）

前端程序员觉得 C++ 是足够快的，V8 觉得 C++ 还不够快，所以 V8 内部使用 CodeStubAssembler 语言，在上文 MicrotaskQueue 的基础上，进一步优化了 microtask 队列的性能。事实上，Javascript 的内置函数，大多都是用 CodeStubAssembler 实现的。CodeStubAssembler 版本的 EnqueueMicrotask [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/builtins/builtins-microtask-queue-gen.cc#474)：


```c++
TF_BUILTIN(EnqueueMicrotask, MicrotaskQueueBuiltinsAssembler) {
  TNode<Microtask> microtask =
      UncheckedCast<Microtask>(Parameter(Descriptor::kMicrotask));
  TNode<Context> context = CAST(Parameter(Descriptor::kContext));
  TNode<Context> native_context = LoadNativeContext(context);
  TNode<RawPtrT> microtask_queue = GetMicrotaskQueue(native_context);

  // 下面的 4 个变量，本质上还是 C++ 版本 MicrotaskQueue 里的同名变量，V8 使用对象 + 偏移量的方式，操作 C++ 版本的 MicrotaskQueue 对象
  TNode<RawPtrT> ring_buffer = GetMicrotaskRingBuffer(microtask_queue); 
  TNode<IntPtrT> capacity = GetMicrotaskQueueCapacity(microtask_queue);
  TNode<IntPtrT> size = GetMicrotaskQueueSize(microtask_queue);
  TNode<IntPtrT> start = GetMicrotaskQueueStart(microtask_queue);

  Label if_grow(this, Label::kDeferred);
  GotoIf(IntPtrEqual(size, capacity), &if_grow);

  // |microtask_queue| has an unused slot to store |microtask|.
  {
    // 将 microtask 存入 ring_buffer，这里的代码等同于前文看到的
    // ring_buffer_[(start_ + size_) % capacity_] = microtask.ptr();
    // ++size_; // 个数自增 1
    StoreNoWriteBarrier(MachineType::PointerRepresentation(), ring_buffer,
                        CalculateRingBufferOffset(capacity, start, size),
                        BitcastTaggedToWord(microtask));
    StoreNoWriteBarrier(MachineType::PointerRepresentation(), microtask_queue,
                        IntPtrConstant(MicrotaskQueue::kSizeOffset),
                        IntPtrAdd(size, IntPtrConstant(1)));
    Return(UndefinedConstant());
  }
  // 源码太长，略
}
```

看过了 C++ 版本的 MicrotaskQueue 的实现。再看 CodeStubAssembler 版本中的变量 ring_buffer，capacity，size，start，有没有似曾相识的感觉呢。其实刚才见到的几个变量，底层还是 C++ 版本中的 MicrotaskQueue 的同名变量。本文只分析下面这行代码，其它代码逻辑类似。

```c++
  TNode<RawPtrT> ring_buffer = GetMicrotaskRingBuffer(microtask_queue); 
```

GetMicrotaskRingBuffer 方法[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/builtins/builtins-microtask-queue-gen.cc#61)：

```c++
TNode<RawPtrT> MicrotaskQueueBuiltinsAssembler::GetMicrotaskRingBuffer(
    TNode<RawPtrT> microtask_queue) {
  // CodeStubAssembler::Print("MicrotaskQueueBuiltinsAssembler::GetMicrotaskRingBuffer");
  return UncheckedCast<RawPtrT>(
      Load(MachineType::Pointer(), microtask_queue,
           IntPtrConstant(MicrotaskQueue::kRingBufferOffset)));
}
```

可见 GetMicrotaskRingBuffer 的逻辑是从 microtask_queue 对象上，读偏移量为 MicrotaskQueue::kRingBufferOffset 的字段。
MicrotaskQueue::kRingBufferOffset [定义如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/execution/microtask-queue.cc#22)：

```c++
#define OFFSET_OF(type, field) \
  (reinterpret_cast<intptr_t>(&(reinterpret_cast<type*>(16)->field)) - 16)

const size_t MicrotaskQueue::kRingBufferOffset =
    OFFSET_OF(MicrotaskQueue, ring_buffer_);
```

MicrotaskQueue::kRingBufferOffset 表示 ring_buffer_ 在 MicrotaskQueue 对象上的偏移量。只要有了偏移量，就可以通过非常规手段访问 ring_buffer_，V8 就是这么干的，这也是本节名称奇技淫巧的由来。

遍历 microtask 队列的方法 RunMicrotasks [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/builtins/builtins-microtask-queue-gen.cc#524)：

```c++
TF_BUILTIN(RunMicrotasks, MicrotaskQueueBuiltinsAssembler) {
  // Load the current context from the isolate.
  TNode<Context> current_context = GetCurrentContext();

  TNode<RawPtrT> microtask_queue =
      UncheckedCast<RawPtrT>(Parameter(Descriptor::kMicrotaskQueue));

  Label loop(this), done(this);
  Goto(&loop);
  BIND(&loop);

  TNode<IntPtrT> size = GetMicrotaskQueueSize(microtask_queue);

  // Exit if the queue is empty.
  GotoIf(WordEqual(size, IntPtrConstant(0)), &done); 

  TNode<RawPtrT> ring_buffer = GetMicrotaskRingBuffer(microtask_queue);
  TNode<IntPtrT> capacity = GetMicrotaskQueueCapacity(microtask_queue);
  TNode<IntPtrT> start = GetMicrotaskQueueStart(microtask_queue);

  TNode<IntPtrT> offset =
      CalculateRingBufferOffset(capacity, start, IntPtrConstant(0));
  TNode<RawPtrT> microtask_pointer =
      UncheckedCast<RawPtrT>(Load(MachineType::Pointer(), ring_buffer, offset));
  TNode<Microtask> microtask = CAST(BitcastWordToTagged(microtask_pointer));

  TNode<IntPtrT> new_size = IntPtrSub(size, IntPtrConstant(1));
  TNode<IntPtrT> new_start = WordAnd(IntPtrAdd(start, IntPtrConstant(1)),
                                     IntPtrSub(capacity, IntPtrConstant(1)));

  // Remove |microtask| from |ring_buffer| before running it, since its
  // invocation may add another microtask into |ring_buffer|.
  SetMicrotaskQueueSize(microtask_queue, new_size);
  SetMicrotaskQueueStart(microtask_queue, new_start);

  RunSingleMicrotask(current_context, microtask);
  IncrementFinishedMicrotaskCount(microtask_queue);
  Goto(&loop);

  BIND(&done);
  {
    // Reset the "current microtask" on the isolate.
    StoreRoot(RootIndex::kCurrentMicrotask, UndefinedConstant());
    Return(UndefinedConstant());
  }
}
```

从 RunMicrotasks 源码来看，整体风格类似于汇编语言写 for 循环，BIND(&loop) 相当于循环体，连续从 ring_buffer 里取出一个 microtask 执行，当循环条件不再满足时，跳转到 BIND(&done)。

本节逻辑整理如下：

- C++ 实现了 MicrotaskQueue
- V8 使用 CodeStubAssembler 对 MicrotaskQueue 做了优化，但底层还是 C++ 版本的 MicrotaskQueue 对象
- CodeStubAssembler 通过 MicrotaskQueue 对象 + 相应字段的偏移量，来操作 MicrotaskQueue 对象的字段，如 ring_buffer_，capacity，size，start 等


## async/await

```JavaScript
async function test() {
  let res = await 123456;
  console.log(res)
}

test()
```
本节以上面的简单 JavaScript 代码为例，分析 async/await 的执行机制。
### 生成字节码

生成 await 123456 的字节码的[代码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/interpreter/bytecode-generator.cc#3805)：
```c++
void BytecodeGenerator::BuildSuspendPoint(int position) {
  const int suspend_id = suspend_count_++;

  RegisterList registers = register_allocator()->AllLiveRegisters();

  // Save context, registers, and state. This bytecode then returns the value
  // in the accumulator.
  builder()->SetExpressionPosition(position);
  builder()->SuspendGenerator(generator_object(), registers, suspend_id);

  // Upon resume, we continue here.
  builder()->Bind(generator_jump_table_, suspend_id);

  // Clobbers all registers and sets the accumulator to the
  // [[input_or_debug_pos]] slot of the generator object.
  builder()->ResumeGenerator(generator_object(), registers);
}
```

虽然很难看懂，但配合 V8 生成的字节码，可以互相印证。await 123456 中 123456 是笔者随便写的一个数，目的是为了和字节码对照。

![awaitByte](https://raw.githubusercontent.com/xudale/blog/master/assets/awaitByte.png)

上图为 JavaScript 代码对应的字节码，从上图来看，和 await 对应的字节码主要为 SuspendGenerator 和 ResumeGenerator。从这两个字节码的命名来推测，JavaScript 代码执行遇到 await，是会暂停执行的，事实也是如此，下文分析。

> V8 对 async/await 有专门的处理，async/await 是关键字
>
> async/await 和 generator 共享许多源码，很多文章说 async/await 是 generator 的语法糖，是有一定道理的
### 执行字节码

字节码 SuspendGenerator 的处理函数，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/interpreter/interpreter-generator.cc#3168)：

```c++
IGNITION_HANDLER(SuspendGenerator, InterpreterAssembler) {
  Node* generator = LoadRegisterAtOperandIndex(0);
  TNode<FixedArray> array = CAST(LoadObjectField(
      generator, JSGeneratorObject::kParametersAndRegistersOffset));
  Node* closure = LoadRegister(Register::function_closure());
  Node* context = GetContext();
  RegListNodePair registers = GetRegisterListAtOperandIndex(1);
  Node* suspend_id = BytecodeOperandUImmSmi(3);

  Node* shared =
      LoadObjectField(closure, JSFunction::kSharedFunctionInfoOffset);
  TNode<Int32T> formal_parameter_count = UncheckedCast<Int32T>(
      LoadObjectField(shared, SharedFunctionInfo::kFormalParameterCountOffset,
                      MachineType::Uint16()));

  ExportParametersAndRegisterFile(array, registers, formal_parameter_count);
  StoreObjectField(generator, JSGeneratorObject::kContextOffset, context);
  StoreObjectField(generator, JSGeneratorObject::kContinuationOffset,
                   suspend_id);

  // Store the bytecode offset in the [input_or_debug_pos] field, to be used by
  // the inspector.
  Node* offset = SmiTag(BytecodeOffset());
  StoreObjectField(generator, JSGeneratorObject::kInputOrDebugPosOffset,
                   offset);

  UpdateInterruptBudgetOnReturn();
  Return(GetAccumulator()); // 注意最后一行
}
```

从源码来看，V8 在执行字节码 SuspendGenerator 时，多次调用 StoreObjectField，保存当前的状态。目前还看不出来代码会暂停执行，这里要注意下代码的最后一行。对比 [ResumeGenerator](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/interpreter/interpreter-generator.cc#3246) 的字节码处理函数来看：

```c++
IGNITION_HANDLER(ResumeGenerator, InterpreterAssembler) {
  Node* generator = LoadRegisterAtOperandIndex(0);
  Node* closure = LoadRegister(Register::function_closure());
  RegListNodePair registers = GetRegisterListAtOperandIndex(1);

  Node* shared =
      LoadObjectField(closure, JSFunction::kSharedFunctionInfoOffset);
  TNode<Int32T> formal_parameter_count = UncheckedCast<Int32T>(
      LoadObjectField(shared, SharedFunctionInfo::kFormalParameterCountOffset,
                      MachineType::Uint16()));

  ImportRegisterFile(
      CAST(LoadObjectField(generator,
                           JSGeneratorObject::kParametersAndRegistersOffset)),
      registers, formal_parameter_count);

  // Return the generator's input_or_debug_pos in the accumulator.
  SetAccumulator(
      LoadObjectField(generator, JSGeneratorObject::kInputOrDebugPosOffset));

  Dispatch(); // 注意最后一行
}
```

ResumeGenerator 多次调用 LoadObjectField，恢复之前代码的执行。ResumeGenerator 就像是 SuspendGenerator 的反函数，一个存储当前代码的执行状态，一个恢复当前代码的执行状态。

ResumeGenerator 的最后一行是 Dispatch，Dispatch 的功能是取出下一条要执行的字节码，然后执行，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7-lkgr/src/interpreter/interpreter-assembler.cc#1396)

```c++
Node* InterpreterAssembler::Dispatch() {
  Comment("========= Dispatch");
  DCHECK_IMPLIES(Bytecodes::MakesCallAlongCriticalPath(bytecode_), made_call_);
  Node* target_offset = Advance();
  Node* target_bytecode = LoadBytecode(target_offset);

  if (Bytecodes::IsStarLookahead(bytecode_, operand_scale_)) {
    target_bytecode = StarDispatchLookahead(target_bytecode);
  }
  return DispatchToBytecode(target_bytecode, BytecodeOffset());
}

Node* InterpreterAssembler::Advance() { return Advance(CurrentBytecodeSize()); }
```

因为 ResumeGenerator 的字节码处理函数，最后一行调用了 Dispatch 来读取执行下一个字节码，所以程序执行不会暂停。几乎所有的字节码处理函数，最后都会调用 Dispatch，让程序一江春水向东流，继续执行。而 SuspendGenerator 最后一行没有调用 Dispatch，所以 V8 在执行 await 生成的字节码 SuspendGenerator 时会暂停当前代码的执行，这是 await 可以暂停程序执行的的根本原因。

V8 在执行 await 123456 时产生的 log 如下，下图的 log 包括字节码生成、字节码执行和 microtask 队列：

![awaitlog](https://raw.githubusercontent.com/xudale/blog/master/assets/awaitlog.png)

从 log 的内容可以看出，await 程序暂停后，在遍历 microtask 队列的过程中，程序才恢复执行。

> await 会暂停当前程序的执行，babel 把 async/await 编译成一个含有 switch case 语句的闭包，与 V8 async/await 的真实执行机制相去甚远


## 总结

![microtaskflow](https://raw.githubusercontent.com/xudale/blog/master/assets/microtaskflow.png)












