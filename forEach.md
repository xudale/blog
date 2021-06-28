# Array.prototype.foreach 源码分析

源码涉及 V8 的两个函数：ArrayForEach 和 FastArrayForEach。先调用 ArrayForEach，收集遍历需要的信息，如遍历次数、回调函数、thisArg 等。最后调用 FastArrayForEach 完成核心的遍历逻辑。

## ArrayForEach

Javascript Array.prototype.foreach 实际调用的是 V8 的 ArrayForEach，[ArrayForEach](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-foreach.tq#92) 源码如下：

```c++
// https://tc39.github.io/ecma262/#sec-array.prototype.foreach
transitioning javascript builtin
ArrayForEach(
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
    // Special cases.
    let k: Number = 0;
    try {
      return FastArrayForEach(o, len, callbackfn, thisArg)
          otherwise Bailout;
    } label Bailout(kValue: Smi) deferred {
      k = kValue;
    }
}
```

ArrayForEach 的逻辑很简单，获取数组 o，循环次数 len，回调函数 callbackfn。因为 foreach 方法的第二个参数非必传，thisArg 可能为空。将上面的 4 个变量当做参数传给 FastArrayForEach，[FastArrayForEach](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-foreach.tq#70) 源码如下：

```c++
transitioning macro FastArrayForEach(implicit context: Context)(
    o: JSReceiver, len: Number, callbackfn: Callable, thisArg: JSAny): JSAny
    labels Bailout(Smi) {

  let k: Smi = 0;
  const smiLen = Cast<Smi>(len) otherwise goto Bailout(k);
  const fastO = Cast<FastJSArray>(o) otherwise goto Bailout(k);
  let fastOW = NewFastJSArrayWitness(fastO);

  // 遍历开始
  for (; k < smiLen; k++) {
    // Ensure that we haven't walked beyond a possibly updated length.
    if (k >= fastOW.Get().length) goto Bailout(k);
    const value: JSAny = fastOW.LoadElementNoHole(k)
        otherwise continue;
    Call(context, callbackfn, thisArg, value, k, fastOW.Get());
  }
  // 返回 Undefined 
  // 多希望可以返回原数组啊，这样就可以链式调用了
  return Undefined;
}
```

FastArrayForEach 的核心逻辑是 for 循环，在 for 循环中反复调用 callbackfn，如果 callbackfn 返回的结果可以转为 false，则函数整体返回 false。如果 for 循环结束，说明每个元素都满足 callbackfn 的测试条件，此时执行最后一行代码：return True。

值得注意的一点是，对空数组调用 foreach，永远返回 true。这一点非常不符合直觉，笔者在这总计踩坑两次。如：

```Javascript
const emptyArray = []
emptyArray.foreach(_ => false) // 对空数组调用 foreach，返回 true
```

从整个遍历过程可以看出，foreach 方法并不改变原数组。

以下内容摘自 [mdn](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/foreach)。

> foreach 遍历的元素范围在第一次调用 callback 之前就已确定了。在调用 foreach 之后添加到数组中的元素不会被 callback 访问到。如果数组中存在的元素被更改，则他们传入 callback 的值是 foreach 访问到他们那一刻的值。那些被删除的元素或从来未被赋值的元素将不会被访问到。

> foreach 和数学中的"所有"类似，当所有的元素都符合条件才会返回true。正因如此，若传入一个空数组，无论如何都会返回 true。（这种情况属于无条件正确：正因为一个空集合没有元素，所以它其中的所有元素都符合给定的条件)


## 简易版 foreach

如果不考虑任何边界条件，尽可能模仿 V8 FastArrayforeach 的实现逻辑，foreach 方法可用 Javascript 实现如下：

```Javascript
function foreach(callback, thisArg) {
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

[ecma262:sec-array.prototype.foreach](https://tc39.es/ecma262/#sec-array.prototype.foreach)

[mdn:Array.prototype.foreach](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/foreach)


















