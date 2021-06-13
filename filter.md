# Array.prototype.filter 源码分析

相关代码主要有两个函数，先调用 ArrayFilter，收集遍历需要的信息，如遍历次数、回调函数、获取 thisArg 等。最后调用 FastArrayFilter 完成遍历逻辑，生成新数组。

## ArrayFilter

Javascript Array.prototype.filter 实际调用的是 V8 的 [ArrayFilter](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-filter.tq#149)ArrayFilter 源码如下：

```c++
transitioning javascript builtin
ArrayFilter(
    js-implicit context: NativeContext, receiver: JSAny)(...arguments): JSAny {
  try {

    // o：待遍历的数组
    const o: JSReceiver = ToObject_Inline(context, receiver);

    // len：数组长度
    const len: Number = GetLengthProperty(o);

    // callbackfn：传入的回调函数
    const callbackfn = Cast<Callable>(arguments[0]) otherwise TypeError;

    // thisArg：调用 callbackfn 时传入的 this
    const thisArg: JSAny = arguments[1];
    let output: JSReceiver;

    // Special cases.
    let k: Number = 0;
    let to: Number = 0;
    try {

      try {
        const smiLen: Smi = Cast<Smi>(len) otherwise goto Bailout(k, to);
        const fastOutput =
            Cast<FastJSArray>(output) otherwise goto Bailout(k, to);
        const fastO = Cast<FastJSArray>(o) otherwise goto Bailout(k, to);
        // 调用 FastArrayFilter
        // fastO：等遍历的数组
        // smiLen：遍历次数
        // callbackfn：回调函数
        // thisArg：调用 callbackfn 时传入的 this
        // fastOutput：生成的新数组
        FastArrayFilter(fastO, smiLen, callbackfn, thisArg, fastOutput)
            otherwise Bailout;
        return output;
      } label Bailout(kValue: Number, toValue: Number) deferred {
        k = kValue;
        to = toValue;
      }
    } label SlowSpeciesCreate {
      output = ArraySpeciesCreate(context, receiver, 0);
    }
  } label TypeError deferred {
    ThrowTypeError(MessageTemplate::kCalledNonCallable, arguments[0]);
  }
}
```

Arrayfilter 的逻辑很简单，获取数组 o；循环次数 len；回调函数 callbackfn；因为 filter 方法的第二个参数非必传，thisArg 可能为空。然后将上面 4 个变量当做参数传给 FastArrayfilter。[FastArrayfilter](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-filter.tq#98) 源码如下：

```c++
transitioning macro FastArrayFilter(implicit context: Context)(
    fastO: FastJSArray, len: Smi, callbackfn: Callable, thisArg: JSAny,
    output: FastJSArray) labels
Bailout(Number, Number) {
  let k: Smi = 0;
  let to: Smi = 0;
  let fastOW = NewFastJSArrayWitness(fastO);
  let fastOutputW = NewFastJSArrayWitness(output);

  // 整个 Array.prototype.filter 核心逻辑是下面的循环
  for (; k < len; k++) {
    // 获取第 k 个元素
    const value: JSAny = fastOW.LoadElementNoHole(k) otherwise continue;
    const result: JSAny =
      // 将 thisArg 传给 callbackfn 的 this
      // value, k, fastOW.Get() 对应 callbackfn 接收的 3 个参数
      Call(context, callbackfn, thisArg, value, k, fastOW.Get());
    // ToBoolean 将 result 转成 Boolean 类型
    // 如果转为 true，则执行下面 if 分支
    if (ToBoolean(result)) {
      try {
        // 将 value 放入等返回的数组
        fastOutputW.Push(value) otherwise SlowStore;
      } label SlowStore {
        FastCreateDataProperty(fastOutputW.stable, to, value);
      }
      to = to + 1;
    }
  }
}
```

FastArrayfilter 的核心逻辑是 for 循环，在 for 循环中反复调用 callbackfn，如果 callbackfn 返回 true，将遍历的 value 放入 fastOutputW 数组。下面内容摘自 mdn。

> filter 遍历的元素范围在第一次调用 callback 之前就已经确定了。在调用 filter 之后被添加到数组中的元素不会被 filter 遍历到。如果已经存在的元素被改变了，则他们传入 callback 的值是 filter 遍历到它们那一刻的值。被删除或从来未被赋值的元素不会被遍历到。


## 简易版 filter

如果不考虑任何边界条件，尽可能模仿 V8 FastArrayfilter 的实现逻辑，filter 方法可用 Javascript 实现如下：

```Javascript
function filter(callback, thisArg) {
  const array = this
  const len = array.length
  const result = []
  for (i = 0; i < len; i++) {
    if (callback.call(thisArg, array[i], i, array)) {
      result.push(array[i])
    }
  }
  return result
}
```

## 参考文献

[ecma262:sec-array.prototype.filter](https://tc39.es/ecma262/#sec-array.prototype.filter)

[mdn:Array.prototype.filter](https://developer.mozilla.org/zh-TW/docs/Web/JavaScript/Reference/Global_Objects/Array/filter)


















