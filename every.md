# Array.prototype.every 源码分析

源码涉及 V8 的两个函数：ArrayEvery 和 FastArrayEvery。先调用 ArrayEvery，收集遍历需要的信息，如遍历次数、回调函数、thisArg 等。最后调用 FastArrayEvery 完成核心的查找逻辑。

## ArrayEvery

Javascript Array.prototype.every 实际调用的是 V8 的 ArrayEvery，[ArrayEvery](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-every.tq#111) 源码如下：

```c++
// https://tc39.github.io/ecma262/#sec-array.prototype.every
transitioning javascript builtin
ArrayEvery(
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

    // Special cases.
    try {
      return FastArrayEvery(o, len, callbackfn, thisArg)
          otherwise Bailout;
    } label Bailout(kValue: Smi) deferred {
      return ArrayEveryLoopContinuation(
          o, callbackfn, thisArg, Undefined, o, kValue, len, Undefined);
    }
  } label TypeError deferred {
    ThrowTypeError(MessageTemplate::kCalledNonCallable, arguments[0]);
  }
}
```

ArrayEvery 的逻辑很简单，获取数组 o，循环次数 len，回调函数 callbackfn。因为 every 方法的第二个参数非必传，thisArg 可能为空。将上面的 4 个变量当做参数传给 FastArrayevery，[FastArrayevery](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-every.tq#86) 源码如下：

```c++
transitioning macro FastArrayEvery(implicit context: Context)(
    o: JSReceiver, len: Number, callbackfn: Callable, thisArg: JSAny): JSAny
    labels Bailout(Smi) {
  let k: Smi = 0;
  const smiLen = Cast<Smi>(len) otherwise goto Bailout(k);
  const fastO: FastJSArray = Cast<FastJSArray>(o) otherwise goto Bailout(k);
  let fastOW = NewFastJSArrayWitness(fastO);

  // 核心逻辑开始
  for (; k < smiLen; k++) {
    // 获取当前待遍历元素 value
    const value: JSAny = fastOW.LoadElementNoHole(k) otherwise continue;
    // 调用回调函数 callbackfn
    // thisArg: callbackfn 接收的 this
    // value: callbackfn 的第 1 个参数 value
    // k: callbackfn 的第 2 个参数 index
    // fastOW.Get(): callbackfn 的第 3 个参数 array
    const result: JSAny =
        Call(context, callbackfn, thisArg, value, k, fastOW.Get());
    // 如果 result 为 false，返回 false
    // 否则继续循环
    if (!ToBoolean(result)) {
      return False;
    }
  }
  // 如果为空数组
  // 或者每次调用 callbackfn 都返回 true，则函数返回 true
  return True;
}
```

FastArrayEvery 的核心逻辑是 for 循环，在 for 循环中反复调用 callbackfn，如果 callbackfn 返回的结果可以转为 false，则返回 false。如果 for 循环结束，说明每次调用 callbackfn 返回的都是 true，此时执行最后一行代码：return True。

值得注意的一点是，对空数组调用 every，永远返回 true。如：

```Javascript
const emptyArray = []
emptyArray.every(_ => false) // 对空数组调用 every，返回 true
```

从整个遍历过程可以看出，every 方法并不改变原数组。

以下内容摘自 [mdn](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/every)。

> every 遍历的元素范围在第一次调用 callback 之前就已确定了。在调用 every 之后添加到数组中的元素不会被 callback 访问到。如果数组中存在的元素被更改，则他们传入 callback 的值是 every 访问到他们那一刻的值。那些被删除的元素或从来未被赋值的元素将不会被访问到。

> every 和数学中的"所有"类似，当所有的元素都符合条件才会返回true。正因如此，若传入一个空数组，无论如何都会返回 true。（这种情况属于无条件正确：正因为一个空集合没有元素，所以它其中的所有元素都符合给定的条件)


## 简易版 every

如果不考虑任何边界条件，尽可能模仿 V8 FastArrayevery 的实现逻辑，every 方法可用 Javascript 实现如下：

```Javascript
function every(callback, thisArg) {
  const array = this
  const len = array.length
  for (let i = 0; i < len; i++) {
    if (callback.call(thisArg, array[i], i, array)) {
      continue
    } else {
      return false
    }
  }
  return true
}
```

## 参考文献

[ecma262:sec-array.prototype.every](https://tc39.es/ecma262/#sec-array.prototype.every)

[mdn:Array.prototype.every](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/every)


















