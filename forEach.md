# Array.prototype.forEach 源码分析

源码涉及 V8 的两个函数：ArrayForEach 和 FastArrayForEach。先调用 ArrayForEach，收集遍历需要的信息，如遍历次数、回调函数、thisArg 等。最后调用 FastArrayForEach 完成核心的遍历逻辑。

## ArrayForEach

Javascript Array.prototype.forEach 实际调用的是 V8 的 ArrayForEach，[ArrayForEach](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-foreach.tq#92) 源码如下：

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

    let k: Number = 0;
    try {
      return FastArrayForEach(o, len, callbackfn, thisArg)
          otherwise Bailout;
    } label Bailout(kValue: Smi) deferred {
      k = kValue;
    }
  }
}
```

ArrayForEach 的逻辑很简单，获取数组 o，循环次数 len，回调函数 callbackfn。因为 forEach 方法的第二个参数非必传，thisArg 可能为空。将上面的 4 个变量当做参数传给 FastArrayForEach，[FastArrayForEach](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-foreach.tq#70) 源码如下：

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
    // 获取当前遍历元素
    const value: JSAny = fastOW.LoadElementNoHole(k)
        otherwise continue;
    Call(context, callbackfn, thisArg, value, k, fastOW.Get());
  }
  // 返回 Undefined 
  // 多希望可以返回原数组啊，这样就可以链式调用了
  return Undefined;
}
```

FastArrayForEach 的核心逻辑是 for 循环，在 for 循环中对每一个元素，都调用 callbackfn，但是不要 callbackfn 的返回结果。最后整个函数 return Undefined;




以下内容摘自 [mdn](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach)。

> forEach() 遍历的范围在第一次调用 callback 前就会确定。调用 forEach 后添加到数组中的项不会被 callback 访问到。如果已经存在的值被改变，则传递给 callback 的值是 forEach() 遍历到他们那一刻的值。已删除的项不会被遍历到


## 简易版 forEach

如果不考虑任何边界条件，尽可能模仿 V8 FastArrayForEach 的实现逻辑，forEach 方法可用 Javascript 实现如下：

```Javascript
function forEach(callback, thisArg) {
  const array = this
  const len = array.length
  for (i = 0; i < len; i++) {
    callback.call(thisArg, array[i], i, array)
  }
  return undefined
}
```

## 参考文献

[ecma262:sec-array.prototype.foreach](https://tc39.es/ecma262/#sec-array.prototype.foreach)

[mdn:Array.prototype.forEach](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach)


















