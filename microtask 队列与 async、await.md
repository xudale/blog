# microtask 队列与 async/await 源码分析
本文分析 V8 7.7.1 版本中的 microtask 队列和 async/await 的核心代码。
## microtask 队列

microtask 队列存在于 V8 中，从 Node 和 V8 源码来看，V8 有暴露 microtask 队列相关的方法给 Node。也就是说 Node 可以操作 V8 的 microtask 队列，比如向 microtask 队列新增一个 microtask，或者把 microtask 队列的每一个 microtask 都执行一遍。
### 基础功能

当执行以下代码：
```JavaScript
Promise.resolve('abc').then(res => console.log(res))
```

V8 的 microtask 队列里会新增一个 microtask，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/execution/microtask-queue.cc#100)：

```c++
void MicrotaskQueue::EnqueueMicrotask(Microtask microtask) {
  if (size_ == capacity_) {
    // size_:microtask 队列中 microtask 的个数
    // capacity_:microtask 队列的容量
    intptr_t new_capacity = std::max(kMinimumCapacity, capacity_ << 1); // 得到新容量
    ResizeBuffer(new_capacity); // 开始扩容
  }

  DCHECK_LT(size_, capacity_);
  // ring_buffer_:存放所有的 microtask
  ring_buffer_[(start_ + size_) % capacity_] = microtask.ptr();
  ++size_; // 个数自增 1
}
```
MicrotaskQueue 对象就是 microtask 队列在 V8 中的抽象表示，size_ 表示 microtask 队列中 microtask 的个数，capacity_ 表示 microtask 队列的容量，如果 size_ == capacity_，表示 microtask 队列容量不够，需要扩容。调用 ResizeBuffer 方法给 microtask 队列扩容。

如果 microtask 队列容量足够，则向 ring_buffer_ 中存入当前的 microtask，ring_buffer_ 是一个指针，通过 ring_buffer_ 可以找到所有的 microtask，ring_buffer_ 定义如下：

```c++
Address* ring_buffer_ = nullptr;
```

上文提到的给 microtask 队列扩容的 ResizeBuffer 方法[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/execution/microtask-queue.cc#261)：

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

> microtask 队列存在于 V8 中，V8 有暴露操作 microtask 队列的方法给 Node
>
> microtask 队列的底层数据结构为数组，或者动态数组


### 奇技淫巧（建议跳过）

前端程序员觉得 C++ 是足够快的，V8 觉得 C++ 还不够快，所以 V8 内部使用 CodeStubAssembler，在上文 MicrotaskQueue 的基础上，把 microtask 队列的逻辑，基本又实现了一遍。事实上，Javascript 的内置函数，大多都是用 CodeStubAssembler 实现的。CodeStubAssembler 版本的 EnqueueMicrotask [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-microtask-queue-gen.cc#474)：


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

GetMicrotaskRingBuffer 方法[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/builtins-microtask-queue-gen.cc#61)：

```c++
TNode<RawPtrT> MicrotaskQueueBuiltinsAssembler::GetMicrotaskRingBuffer(
    TNode<RawPtrT> microtask_queue) {
  // CodeStubAssembler::Print("MicrotaskQueueBuiltinsAssembler::GetMicrotaskRingBuffer");
  return UncheckedCast<RawPtrT>(
      Load(MachineType::Pointer(), microtask_queue,
           IntPtrConstant(MicrotaskQueue::kRingBufferOffset)));
}
```

可见 GetMicrotaskRingBuffer 的逻辑是从 microtask_queue 对象上，取现偏移量为 MicrotaskQueue::kRingBufferOffset 的字段。
MicrotaskQueue::kRingBufferOffset [定义如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/execution/microtask-queue.cc#22)：

```c++
#define OFFSET_OF(type, field) \
  (reinterpret_cast<intptr_t>(&(reinterpret_cast<type*>(16)->field)) - 16)

const size_t MicrotaskQueue::kRingBufferOffset =
    OFFSET_OF(MicrotaskQueue, ring_buffer_);
```

MicrotaskQueue::kRingBufferOffset 表示 ring_buffer_ 在 MicrotaskQueue 对象上的偏移量。只要有了偏移量，就可以通过非常规手段访问 ring_buffer_，V8 就是这么干的，这也是本节名称奇技淫巧的由来。
本节逻辑整理如下：

- C++ 实现了 MicrotaskQueue
- V8 使用 CodeStubAssembler 对 MicrotaskQueue 做了优化，但底层还是 C++ 版本的 MicrotaskQueue 对象
- CodeStubAssembler 通过 MicrotaskQueue 对象 + 相应字段的偏移量，来进行 microtask 相关的操作，如 microtask 进入队列，执行 microtask 队列中全部的 microtask

## async/await

### 生成字节码
### 执行字节码




## 总结

Boolean、== 和 === 在 V8 中是 3 段互相独立的逻辑，不可混淆。

> 随堂小测验：
> 
> Boolean('0') // true，因为 '0' 是字符串且长度大于 0
>
> Boolean('') // false，因为 '' 是空字符串且长度为 0
>
> null == undefined // true
>
> null == '' // false，null 与 undefined 以外的绝大多数类型都不相等
>
> null == '0' // false
>
> null == false // false
>
> null == document.all // true，document.all 不按套路出牌
>
> undefined == document.all // true
>
> Boolean(document.all) // false
>
> NaN == NaN // false，NaN 和谁都不相等










