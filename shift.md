# Array.prototype.shift 源码分析

## TryFastArrayShift

Javascript Array.prototype.shift 实际调用的是 V8 的 TryFastArrayShift[TryFastArrayShift](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-shift.tq#8) 源码如下：

```c++
macro TryFastArrayShift(implicit context: Context)(receiver: JSAny): JSAny
    labels Slow, Runtime {
  const array: FastJSArray = Cast<FastJSArray>(receiver) otherwise Slow;

  // witness 相当于对数组又包了一层
  let witness = NewFastJSArrayWitness(array);

  // 如果是空数组，则返回 Undefined
  if (array.length == 0) {
    return Undefined;
  }

  // 长度 - 1
  const newLength = array.length - 1;

  // 获取第 0 个元素
  const result = witness.LoadElementOrUndefined(0);

  // 修改数组长度为原长度 - 1
  witness.ChangeLength(newLength);
  // 把 index 从 1 开始的所有数组元素
  // 复制到 index 0 开始的内存
  witness.MoveElements(0, 1, Convert<intptr>(newLength));
  // 将改变后的数组的最后一个元素置为空
  witness.StoreHole(newLength);
  // 返回原数组第 0 个元素
  return result;
}
```

TryFastArrayShift 的逻辑是取出 index 为 0 的元素，做为函数的返回结果。并将 index 为 1 的元素，复制到 index 0，index 为 2 的元素，复制到 index 1，依此类推。调用 ChangeLength 方法改变数组的长度，调用 StoreHole 将数组最后一个元素清空。笔者最感兴趣的是 MoveElements，因为只有这个方法从方法名字上看不出来到底做了什么工作。

## MoveElements

MoveElements 底层调用了 CodeStubAssembler::MoveElements，CodeStubAssembler::MoveElements 的功能是内存复制，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/codegen/code-stub-assembler.cc#4594)：

```c++
void CodeStubAssembler::MoveElements(ElementsKind kind,
                                     TNode<FixedArrayBase> elements,
                                     TNode<IntPtrT> dst_index,
                                     TNode<IntPtrT> src_index,
                                     TNode<IntPtrT> length) {
  // 计算需要复制的字节数
  const TNode<IntPtrT> source_byte_length =
      IntPtrMul(length, IntPtrConstant(ElementsKindToByteSize(kind)));
  static const int32_t fa_base_data_offset =
      FixedArrayBase::kHeaderSize - kHeapObjectTag;
  TNode<IntPtrT> elements_intptr = BitcastTaggedToWord(elements);
  // 内存复制的目的地址
  TNode<IntPtrT> target_data_ptr =
      IntPtrAdd(elements_intptr,
                ElementOffsetFromIndex(dst_index, kind, fa_base_data_offset));
  // 内存复制的源地址
  TNode<IntPtrT> source_data_ptr =
      IntPtrAdd(elements_intptr,
                ElementOffsetFromIndex(src_index, kind, fa_base_data_offset));
  // 获取 memmove 函数
  TNode<ExternalReference> memmove =
      ExternalConstant(ExternalReference::libc_memmove_function());
  // 调用 memmove 函数，传的参数是上面得到的目的地址，源地址，待复制的字节数
  CallCFunction(memmove, // 函数地址
                MachineType::Pointer(), // memmove 的返回类型
                // 传给 memmove 的 3 个参数
                std::make_pair(MachineType::Pointer(), target_data_ptr), 
                std::make_pair(MachineType::Pointer(), source_data_ptr),
                std::make_pair(MachineType::UintPtr(), source_byte_length));
}
```

memmove 函数底层调用的是 C 标准库函数 memmove，C 库函数 memmove 从源地址复制 n 个字符到目的地址。[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/codegen/external-reference.cc#754)：

```c++
void* libc_memmove(void* dest, const void* src, size_t n) {
  return memmove(dest, src, n);
}

FUNCTION_REFERENCE(libc_memmove_function, libc_memmove)
```

> shift 的底层调用了 C 语言的 memmove 做内存复制，复杂度 O(n)。对于超长数组，慎重调用 shift。

## 简易版 shift

```C++
function shift() {
  const array = this
  const firstElement = array[0]
  for (let i = 1; i < array.length; i++) {
    array[i - 1] = array[i]
  }
  array.length = array.length - 1
  return firstElement
}
```

## 参考文献

[ecma262:sec-array.prototype.shift](https://tc39.es/ecma262/#sec-array.prototype.shift)

[mdn:Array.prototype.shift](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/shift)


















