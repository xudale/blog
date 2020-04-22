# microtask 队列与 async/await 源码分析
本文分析 V8 7.7.1 版本中的 microtask 队列和 async/await 的核心代码。
## microtask 队列

microtask 队列存在于 V8 中，从 Node 和 V8 源码来看，V8 有暴露 microtask 队列相关的方法给 Node。也就是说 Node 可以操作 V8 的 microtask 队列，比如向 microtask 队列新增一个 microtask，或者把 microtask 队列的每一个 microtask 都执行一遍。
### 基础逻辑

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
在 JavaScript 中，Boolean 函数有两种调用方式，一种是函数式调用：

```JavaScript
Boolean('test') // true
```
一种是构造函数式调用:

```JavaScript
new Boolean('test') // Boolean {true}
```
无论是哪种调用方式，在 V8 中都是由同一个函数处理，Boolean 函数由 [Torque](https://v8.dev/docs/torque-builtins) 实现，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/boolean.tq#22)：

```c++
BooleanConstructor(context: Context, receiver: Object, ...arguments): Object {
  const value = SelectBooleanConstant(ToBoolean(arguments[0])); // 将参数转为 true 或 false
  const newTarget = Parameter(NEW_TARGET_INDEX); // 如果是构造函数式调用 new Boolean()，newTarget 一定有值
  if (newTarget == Undefined) {
    // 如果是函数式调用，此处返回
    return value;
  }
  // 如果是构造函数式调用，执行下面逻辑，建议先忽略
  const target = UnsafeCast<JSFunction>(Parameter(TARGET_INDEX));
  const map = GetDerivedMap(target, UnsafeCast<JSReceiver>(newTarget));
  let properties = kEmptyFixedArray;
  if (IsDictionaryMap(map)) {
    properties = AllocateNameDictionary(kNameDictionaryInitialCapacity);
  }
  const obj = UnsafeCast<JSValue>(AllocateJSObjectFromMap(
      map, properties, kEmptyFixedArray, kNone, kWithSlackTracking));
  obj.value = value;
  return obj;
}
```

BooleanConstructor 函数的逻辑很简单，通过 ToBoolean(arguments[0]) 将参数转为 true 或 false，如果是函数式调用，立刻返回结果，这也是日常开发中常见的情况。

ToBoolean [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/base.tq#2784)：

```c++
macro ToBoolean(obj: Object): bool {
  if (BranchIfToBooleanIsTrue(obj)) {
    return true;
  } else {
    return false;
  }
}
```

从源码来看 ToBoolean 函数只是在 BranchIfToBooleanIsTrue 外面包了一层而已，ToBoolean 源码也是由 Torque 实现，Torque 语法类似 Typescript。重头戏 BranchIfToBooleanIsTrue 由 CodeStubAssembler 实现，CodeStubAssembler 整体语法类似汇编，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/codegen/code-stub-assembler.cc#1328):

```c++
void CodeStubAssembler::BranchIfToBooleanIsTrue(Node* value, Label* if_true,
                                                Label* if_false) {
  // 定义标号
  Label if_smi(this), if_notsmi(this), if_heapnumber(this, Label::kDeferred),
      if_bigint(this, Label::kDeferred);
  // Rule out false {value}. 如果参数是 false，直接返回 false
  GotoIf(WordEqual(value, FalseConstant()), if_false);
  // Check if {value} is a Smi or a HeapObject.
  // 根据 value 是否是 smi，跳转到不同的分支
  Branch(TaggedIsSmi(value), &if_smi, &if_notsmi);
  BIND(&if_smi);
  {
    // The {value} is a Smi, only need to check against zero.
    // 如果 value 等于 0，返回 false，其它返回 true
    BranchIfSmiEqual(CAST(value), SmiConstant(0), if_false, if_true);
  }
  BIND(&if_notsmi);
  {
    // Check if {value} is the empty string.
    // 如果是空字符串，直接返回 false
    GotoIf(IsEmptyString(value), if_false);
    // The {value} is a HeapObject, load its map.
    Node* value_map = LoadMap(value);
    // Only null, undefined and document.all have the undetectable bit set,
    // so we can return false immediately when that bit is set.
    GotoIf(IsUndetectableMap(value_map), if_false);
    // We still need to handle numbers specially, but all other {value}s
    // that make it here yield true.
    GotoIf(IsHeapNumberMap(value_map), &if_heapnumber);
    Branch(IsBigInt(value), &if_bigint, if_true);
    BIND(&if_heapnumber);
    {
      // Load the floating point value of {value}.
      Node* value_value = LoadObjectField(value, HeapNumber::kValueOffset,
                                          MachineType::Float64());
      // Check if the floating point {value} is neither 0.0, -0.0 nor NaN.
      Branch(Float64LessThan(Float64Constant(0.0), Float64Abs(value_value)),
             if_true, if_false);
    }
    BIND(&if_bigint);
    {
      TNode<BigInt> bigint = CAST(value);
      TNode<Word32T> bitfield = LoadBigIntBitfield(bigint);
      TNode<Uint32T> length = DecodeWord32<BigIntBase::LengthBits>(bitfield);
      Branch(Word32Equal(length, Int32Constant(0)), if_false, if_true);
    }
  }
}
```
Label 类似汇编语言的标号，GotoIf 和 Branch 类似汇编语言的条件跳转，smi 是 Small Integer 的缩写，代码逻辑有些冗长。对照 ECMAScript Spec 可以看出代码逻辑与 Spec 的定义是一致的。Spec 中 ToBoolean 定义如下：

![ToBoolean](https://raw.githubusercontent.com/xudale/blog/master/assets/ToBoolean.png)

比如 Spec 中明确规定对于 Number，Return false if argument is +0, −0, or NaN; otherwise return true，V8 对应源码是：

```c++
BIND(&if_smi);
{
  // The {value} is a Smi, only need to check against zero.
  // 如果 value 等于 0，返回 false，其它返回 true
  BranchIfSmiEqual(CAST(value), SmiConstant(0), if_false, if_true);
}
```

> V8 中 Boolean 的实现逻辑看似冗长，其实和 ECMAScript Spec 的定义是一一对应的
> 
> 在 JavaScript 中无论是 Boolean 还是 new Boolean()，在 V8 中执行的是同一个函数，只是分支和返回值不同


## == 运算符

V8 [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/codegen/code-stub-assembler.cc#11758)

```c++
// ES6 section 7.2.12 Abstract Equality Comparison
Node* CodeStubAssembler::Equal(Node* left, Node* right, Node* context,
                               Variable* var_type_feedback) {
  // 前面省略几十行
  Label if_notsame(this);
  GotoIf(WordNotEqual(left, right), &if_notsame);
  {
    // {left} and {right} reference the exact same value, yet we need special
    // treatment for HeapNumber, as NaN is not equal to NaN.
    GenerateEqual_Same(left, &if_equal, &if_notequal, var_type_feedback);
  }
  BIND(&if_notsame);
  Label if_left_smi(this), if_left_not_smi(this);
  Branch(TaggedIsSmi(left), &if_left_smi, &if_left_not_smi);
  BIND(&if_left_smi);
  {
    Label if_right_smi(this), if_right_not_smi(this);
    Branch(TaggedIsSmi(right), &if_right_smi, &if_right_not_smi);
    BIND(&if_right_smi);
    {
      // We have already checked for {left} and {right} being the same value,
      // so when we get here they must be different Smis.
      CombineFeedback(var_type_feedback,
                      CompareOperationFeedback::kSignedSmall);
      Goto(&if_notequal);
    }
  }
  // 后面省略几百行
}

```

JavaScript 中的 == 运算符，V8 中相应的源码大约有 400 行，本文不做完全分析，看 ECMAScript Spec 相关的定义后再看源码会简单一些：
![AbstractEquality](https://raw.githubusercontent.com/xudale/blog/master/assets/AbstractEquality.png)
从 Spec 来看，大多数情况下，== 运算符是把左右两个操作数都转换成 Number 后，再做比较。虽然 Spec 只定义了少数 case，然而 V8 却需要 400 行代码去实现。如果 Spec 定义了全部可能的 case，需要几千行代码来实现，得不偿失。个人认为这是 Spec 未定义 == 运算符可能出现的所有 case 的原因。
> 面试中遇到 x == y 应该怎么回答？
> 
> 1.JavaScript 有 8 种数据类型，即 Number、String、Boolean、Null、Undefined、Object、Symbol 和 BigInt，两两组合，有 8 * 8 = 64 种 case。ECMAScript Spec 只定义了 10 几种 case，其它 40 多种 case 一律返回 false。所以遇到类似题目只要蒙 false，正确率可达 70%
>
> 2.null 与 undefined 相等，反之也成立。null 或 undefined 与其它类型绝大多数情况下都不相等，遇到 null/undefined == 0/false/'0' 之类的问题，请回答 false，此时正确率可达 80%
>
> 3.明显可以转换为 Number 的情况，如 1 == true/'1'，把 == 左右两边的值都转换成 Number 再比较，此时正确率可达 90%。剩下的情况本文已放弃，看官继续努力。总之，只要回答 false 至少就有 80% 的正确率

## === 运算符

=== 运算符最为安全稳妥，因为坑少，只截取一小部分[伪代码](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/codegen/code-stub-assembler.cc#12155)：

```c++
Node* CodeStubAssembler::StrictEqual(Node* lhs, Node* rhs,
                                     Variable* var_type_feedback) {
  // Pseudo-code for the algorithm below:
  //
  // if (lhs == rhs) {
  //   if (lhs->IsHeapNumber()) return HeapNumber::cast(lhs)->value() != NaN;
  //   return true;
  // }
}
```

伪代码描述的情况是如果左右两个操作数在 C++ 层面相等，但其中一个操作数是 NaN，则返回 false，即

```JavaScript
NaN === NaN // false
```

这应该是 === 运算符唯一的一个坑点。

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










