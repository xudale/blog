# 聊聊 Boolean、==和===
业务开发，踩坑多次，本文将从 V8 源码分析 Boolean、==和===。
## Boolean
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
  // 如果是构造函数式调用，执行下面逻辑，建议不看
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

BooleanConstructor 函数的逻辑很简单，通过 ToBoolean(arguments[0]) 将参数转为 true 或 false，如果是函数式调用，立刻返回，这也是日常开发中常见的情况。

ToBoolean 还是由 Torque 实现，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/builtins/base.tq#2784)：

```c++
macro ToBoolean(obj: Object): bool {
  if (BranchIfToBooleanIsTrue(obj)) {
    return true;
  } else {
    return false;
  }
}
```

从源码来看 ToBoolean 函数只是在 BranchIfToBooleanIsTrue 外面包了一层而已，ToBoolean 源码也是由 Torque 实现，虽然我们都没有学习过 Torque 语言，但只要有任何一门静态类型语言的基础，基本可以看懂。重头戏 BranchIfToBooleanIsTrue 由 CodeStubAssembler 实现，CodeStubAssembler 整体语法类似汇编，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/7.7.1/src/codegen/code-stub-assembler.cc#1328):

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
Label 类似汇编语言的标号，GotoIf 和 Branch 类似汇编语言的条件跳转，smi 是 Small Integer 的缩写，代码逻辑有些冗长。对照 ECMAScript Spec 可以看出代码逻辑与 Spec 的定义是一致的。Spec 中 ToBoolean 规范如下：

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


## ==
## ===




[V8 源码](https://cs.chromium.org/chromium/src/v8/?g=0)可在浏览器中查看，这个网站在代码浏览与检索方面的功能十分强大，可以快速的查看 C++ 变量的定义和引用。缺点是不能查看 V8 某个版本或某个 git tag 的源码，但依然强烈推荐。如果想要查看 V8 某个 tag 的源码，可以访问 [v8.git](https://chromium.googlesource.com/v8/v8.git) 。如果想要在本地编译 V8 源码，在参考 [V8 官方文档](https://v8.dev/docs/build-gn)的基础上，还要注意墙的问题，浏览器能访问 google 不代表终端也能访问 google，终端也需要设置代理。








