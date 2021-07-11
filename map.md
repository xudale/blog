# Array.prototype.map 源码分析

源码涉及 V8 的两个函数：ArrayMap 和 FastArrayMap。先调用 ArrayMap，收集遍历需要的信息，如遍历次数、回调函数、thisArg 等。最后调用 FastArrayMap 完成核心的遍历逻辑。

## ArrayMap

Javascript Array.prototype.map 实际调用的是 V8 的 ArrayMap，[ArrayMap](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-map.tq#229) 源码如下：

```c++
// https://tc39.github.io/ecma262/#sec-array.prototype.map
transitioning javascript builtin
ArrayMap(
    js-implicit context: NativeContext, receiver: JSAny)(...arguments): JSAny {
  try {
    // 获取数组
    const o: JSReceiver = ToObject_Inline(context, receiver);

    // 获取数组长度
    const len: Number = GetLengthProperty(o);

    // 获取回调函数
    const callbackfn = Cast<Callable>(arguments[0]) otherwise TypeError;

    // 获取 thisArg，可选
    const thisArg: JSAny = arguments[1];

    let array: JSReceiver;
    let k: Number = 0;
    try {

      const o: FastJSArrayForRead = Cast<FastJSArrayForRead>(receiver)
          otherwise SlowSpeciesCreate;
      const smiLength: Smi = Cast<Smi>(len)
          otherwise SlowSpeciesCreate;
      // FastArrayMap 承担 map 的主要逻辑
      return FastArrayMap(o, smiLength, callbackfn, thisArg)
          otherwise Bailout;
    }
}
```

ArrayMap 的逻辑很简单，获取数组 o，循环次数 len，回调函数 callbackfn。因为 map 方法的第二个参数非必传，thisArg 可能为空。将上面的 4 个变量当做参数传给 FastArrayMap。

## FastArrayMap

[FastArrayMap](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-map.tq#192) 源码如下：

```c++
transitioning macro FastArrayMap(implicit context: Context)(
    fastO: FastJSArrayForRead, len: Smi, callbackfn: Callable,
    thisArg: JSAny): JSArray
    labels Bailout(JSArray, Smi) {
  let k: Smi = 0;
  let fastOW = NewFastJSArrayForReadWitness(fastO);
  // 新创建一个定长数组，用于存返回结果
  let vector = NewVector(len);

  try {
    for (; k < len; k++) {
      try {
        const value: JSAny = fastOW.LoadElementNoHole(k)
            otherwise FoundHole;
        const result: JSAny =
            Call(context, callbackfn, thisArg, value, k, fastOW.Get());
        // 将本次的结果 result 存入 vector    
        vector.StoreResult(k, result);
      }
    }
  }
  // 循环结束，返回 vector
  return vector.CreateJSArray(len);
}
```

FastArrayMap 的核心逻辑是先创建一个数组 vector 用于存放返回结果，然后开始 for 循环，在 for 循环中对每一个元素，都调用 callbackfn，并将当前 callbackfn 的返回值 result 存入 vector，for 循环结束后返回 vector。

## 其它相关源码

FastArrayMap 中调用 NewVector 生成一个定长数组，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-map.tq#179)：

```c++
macro NewVector(implicit context: Context)(length: Smi): Vector {
  const fixedArray = length > 0 ?
      AllocateFixedArrayWithHoles(
          SmiUntag(length), AllocationFlag::kAllowLargeObjectAllocation) :
      kEmptyFixedArray;
  return Vector{
    fixedArray,
    onlySmis: true,
    onlyNumbers: true,
    skippedElements: false
  };
}
```

NewVector 返回 Vector 对象，上一节为了便于叙述，称 Vector 是数组，确切的说，Vector 是一个包含数组(fixedArray)的对象，onlySmis 和 onlyNumbers 表示数组的类型，V8 数组有几十种类型，具体参考 [Mathias Bynens - V8 internals for JavaScript developers](https://www.youtube.com/watch?v=m9cTaYI95Zc&t=33s)。在本文涉及的范围内，如果一个数组所有的元素都是 SMI，则 onlySmis 为 true，如果一个数组的所有元素都是浮点数，则 onlyNumbers 为 true。Vector [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-map.tq#99)：

```c++
struct Vector {
  macro CreateJSArray(implicit context: Context)(validLength: Smi): JSArray {
    // 源码太长，略，感兴趣的同学请参考
    // https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-map.tq#104
  }

  macro StoreResult(implicit context: Context)(index: Smi, result: JSAny) {
    typeswitch (result) {
      case (s: Smi): {
        this.fixedArray.objects[index] = s;
      }
      case (s: HeapNumber): {
        // 如果发现了浮点数，则 onlySmis 为 false
        this.onlySmis = false;
        this.fixedArray.objects[index] = s;
      }
      case (s: JSAnyNotNumber): {
        // 如果既不是 SMI，也不是浮点数
        // onlySmis 和 onlyNumbers 皆为 false
        this.onlySmis = false;
        this.onlyNumbers = false;
        this.fixedArray.objects[index] = s;
      }
    }
  }

  fixedArray: FixedArray;
  onlySmis: bool;         // initially true.
  onlyNumbers: bool;      // initially true.
  skippedElements: bool;  // initially false.
}
```

FastArrayMap 把每一个回调函数的返回结果都存入 vector，调用的是 StoreResult，StoreResult 的逻辑是把元素存入 fixedArray 字段，同时判断一下当前数组的类型。

FastArrayMap 的最后一行是 return vector.CreateJSArray(len)，CreateJSArray 的逻辑是返回 JS 代码能访问的那种数组，同时根据数组类型，做了下优化。


以下内容摘自 [mdn](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/map)。

> map 方法处理数组元素的范围是在 callback 方法第一次调用之前就已经确定了。调用 map 方法之后追加的数组元素不会被 callback 访问。如果存在的数组元素改变了，那么传给 callback 的值是 map 访问该元素时的值。在 map 函数调用后但在访问该元素前，该元素被删除的话，则无法被访问到


## 简易版 map

如果不考虑任何边界条件，尽可能模仿 V8 FastArrayMap 的实现逻辑，map 方法可用 Javascript 实现如下：

```Javascript
function map(callback, thisArg) {
  const array = this
  const len = array.length
  const result = []
  for (i = 0; i < len; i++) {
    result.push(callback.call(thisArg, array[i], i, array))
  }
  return result
}
```

## 经典/讨厌的面试题：

```Javascript
["1", "2", "3"].map(parseInt); // 返回 [1, NaN, NaN]
```

自从 18 年初见起，这道题，笔者从来没有做对过，做错的原因是对 parseInt("1", 0) 的理解有误。parseInt 的第 2 个参数 radix，如果传了 0，最终效果相当于没传或者传了 10，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/number.tq#253)：

```c++
transitioning builtin ParseInt(implicit context: Context)(
    input: JSAny, radix: JSAny): Number {
  try {
    // 看到这一行已经足够处理那道面试题了
    // radix 是第 2 个参数，从下面的 if 判断可见
    // radix 参数不传、传 10 和传 0 三者效果差不多
    if (radix != Undefined && !TaggedEqual(radix, SmiConstant(10)) &&
        !TaggedEqual(radix, SmiConstant(0))) {
      goto CallRuntime;
    }
  }
}
```

## 参考文献

[ecma262:sec-array.prototype.map](https://tc39.es/ecma262/#sec-array.prototype.map)

[mdn:Array.prototype.map](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/map)

[Mathias Bynens - V8 internals for JavaScript developers](https://www.youtube.com/watch?v=m9cTaYI95Zc&t=33s)


















