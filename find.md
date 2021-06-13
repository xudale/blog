# Array.prototype.find 源码分析

源码涉及 V8 的两个函数：ArrayPrototypeFind 和 FastArrayFind。先调用 ArrayPrototypeFind，收集遍历需要的信息，如遍历次数、回调函数、thisArg 等。最后调用 FastArrayFind 完成核心的查找逻辑，遍历数组，将数组中的每一个元素都当做参数传入回调函数，直到回调函数返回 true，返回当前遍历的数组元素。

## ArrayPrototypeFind

Javascript Array.prototype.find 实际调用的是 V8 的 ArrayPrototypeFind，[ArrayPrototypeFind](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-find.tq#120) 源码如下：

```c++
transitioning javascript builtin
ArrayPrototypeFind(
    js-implicit context: NativeContext, receiver: JSAny)(...arguments): JSAny {
  try {

    // o：待遍历的数组
    const o: JSReceiver = ToObject_Inline(context, receiver);

    // len：数组长度
    const len: Number = GetLengthProperty(o);

    // callbackfn：传入的回调函数
    const callbackfn = Cast<Callable>(arguments[0]) otherwise NotCallableError;

    // thisArg：调用 callbackfn 时传入的 this，可选参数
    const thisArg: JSAny = arguments[1];

    try {
      return FastArrayFind(o, len, callbackfn, thisArg)
          otherwise Bailout;
    } label Bailout(k: Smi) deferred {
      return ArrayFindLoopContinuation(o, callbackfn, thisArg, o, k, len);
    }
  }
}
```

ArrayPrototypeFind 的逻辑很简单，获取数组 o，循环次数 len，回调函数 callbackfn。因为 find 方法的第二个参数非必传，thisArg 可能为空。将上面的 4 个变量当做参数传给 FastArrayfind，[FastArrayfind](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-find.tq#93) 源码如下：

```c++
transitioning macro FastArrayFind(implicit context: Context)(
    o: JSReceiver, len: Number, callbackfn: Callable, thisArg: JSAny): JSAny
    labels Bailout(Smi) {
  let k: Smi = 0;
  const smiLen = Cast<Smi>(len) otherwise goto Bailout(k);
  const fastO = Cast<FastJSArray>(o) otherwise goto Bailout(k);
  let fastOW = NewFastJSArrayWitness(fastO);

  // 核心逻辑在此
  for (; k < smiLen; k++) {
    // 获取第 k 个元素
    const value: JSAny = fastOW.LoadElementOrUndefined(k);
    // 这里的 callbackfn 就是 find 接收的第 1 个参数，调用之
    // 将 thisArg 传给 callbackfn 的 this
    // value, k, fastOW.Get() 对应 callbackfn 接收的 3 个参数：value，index，array
    const testResult: JSAny =
        Call(context, callbackfn, thisArg, value, k, fastOW.Get());
    // 如果结果 testResult 可以转换为 true，则执行下面 if 分支
    // 并返回当前遍历的元素：value
    if (ToBoolean(testResult)) {
      return value;
    }
  }
  // 执行到这里，说明没有找到使 callbackfn 能返回 true 的元素
  return Undefined;
}
```

FastArrayfind 的核心逻辑是 for 循环，在 for 循环中反复调用 callbackfn，如果 callbackfn 返回的结果可以转为 true，则返回当前遍历的元素 value，函数结束。如果 for 循环结束，说明没有找到能使 callbackfn 返回 true 的元素，此时执行最后一行代码：return Undefined。

以下内容摘自 [mdn](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/find)。

> 在第一次调用 callback 函数时会确定元素的索引范围，因此在 find 方法开始执行之后添加到数组的新元素将不会被 callback 函数访问到。如果数组中一个尚未被 callback 函数访问到的元素的值被 callback 函数所改变，那么当 callback 函数访问到它时，它的值是将是根据它在数组中的索引所访问到的当前值。被删除的元素仍旧会被访问到，但是其值已经是 undefined 了


## 简易版 find

如果不考虑任何边界条件，尽可能模仿 V8 FastArrayfind 的实现逻辑，find 方法可用 Javascript 实现如下：

```Javascript
function find(callback, thisArg) {
  const array = this
  const len = array.length
  for (i = 0; i < len; i++) {
    if (callback.call(thisArg, array[i], i, array)) {
      return array[i]
    }
  }
}
```

## 参考文献

[ecma262:sec-array.prototype.find](https://tc39.es/ecma262/#sec-array.prototype.find)

[mdn:Array.prototype.find](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/find)


















