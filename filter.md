# Array.prototype.filter 源码分析

相关代码主要有两个函数，ArrayFilter 收集遍历需要的信息，如遍历次数，回调函数，获取 thisArg 等。FastArrayFilter 完成遍历逻辑，生成返回的新数组。

## ArrayFilter

Javascript Array.prototype.filter 实际调用的是 V8 的 [ArrayFilter](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-filter.tq#149)ArrayFilter 源码如下：

```c++
transitioning javascript builtin
Arrayfilter(
    js-implicit context: NativeContext, receiver: JSAny)(...arguments): JSAny {
  try {
    // o 相当于待遍历的数组
    const o: JSReceiver = ToObject_Inline(context, receiver);

    // 获取数组(也可能是类数组)长度，这个长度也就是循环次数
    // 从这里可以看到，循环次数在数组被遍历前确定
    // 如果在遍历过程中增加或删除数组元素，循环次数不变
    const len: Number = GetLengthProperty(o);

    // 如果调用 filter 方法，却没有传参，报错
    if (arguments.length == 0) {
      goto NoCallableError;
    }
    // 获取 filter 方法的第一个参数，回调函数
    const callbackfn = Cast<Callable>(arguments[0]) otherwise NoCallableError;

    // 试图获取 filter 方法的第二个参数，可以为空
    const initialValue: JSAny|TheHole =
        arguments.length > 1 ? arguments[1] : TheHole;
    // 至此
    // o 是原数组(可能是类数组的对象)
    // len 数组长度，也是循环次数
    // callbackfn 是 filter 方法接收的第一个参数，回调函数
    // initialValue 是 filter 方法接收的第二个参数，初始值，可以为空
    try {
      // 主要逻辑在 FastArrayfilter
      return FastArrayfilter(o, len, callbackfn, initialValue)
          otherwise Bailout;
    } label Bailout(value: Number, accumulator: JSAny|TheHole) {
      return ArrayfilterLoopContinuation(
          o, callbackfn, accumulator, o, value, len);
    }
  } label NoCallableError deferred {
    ThrowTypeError(MessageTemplate::kCalledNonCallable, arguments[0]);
  }
}
```

Arrayfilter 的逻辑很简单，获取数组 o；循环次数 len；回调函数 callbackfn；初始值 initialValue；因为 filter 方法的第二个参数非必传，initialValue 可以为空。然后将上面 4 个变量当做参数传给 FastArrayfilter。[FastArrayfilter](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/9.0-lkgr/src/builtins/array-filter.tq#118) 源码如下：

```c++
transitioning macro FastArrayfilter(implicit context: Context)(
    o: JSReceiver, len: Number, callbackfn: Callable,
    initialAccumulator: JSAny|TheHole): JSAny
    labels Bailout(Number, JSAny | TheHole) {
  const k = 0;
  let accumulator = initialAccumulator;
  const fastO =
      Cast<FastJSArrayForRead>(o) otherwise goto Bailout(k, accumulator);
  let fastOW = NewFastJSArrayForReadWitness(fastO);

  // 循环次数 len，在循环前就已确定
  // filter 本质还是 for 循环
  for (let k: Smi = 0; k < len; k++) {
    // 取出待遍历的数组元素
    const value: JSAny = fastOW.LoadElementNoHole(k) otherwise continue;
    typeswitch (accumulator) {
      case (TheHole): {
        accumulator = value;
      }
      case (accumulatorNotHole: JSAny): {
        // 这里的 callbackfn 是 filter 方法接收的回调函数，调用之
        accumulator = Call(
            context, callbackfn, Undefined, accumulatorNotHole, value, k,
            fastOW.Get());
      }
    }
  }
  typeswitch (accumulator) {
    case (TheHole): {
      ThrowTypeError(
          MessageTemplate::kfilterNoInitial, 'Array.prototype.filter');
    }
    case (accumulator: JSAny): {
      // 返回结果
      return accumulator;
    }
  }
}
```

FastArrayfilter 的核心逻辑是 for 循环，在 for 循环中反复调用 callbackfn，并将每一次返回的结果，做为下一次调用 callbackfn 的参数。

```c++
for (let k: Smi = 0; k < len; k++) {
  const value: JSAny = fastOW.LoadElementNoHole(k) otherwise continue;
  typeswitch (accumulator) {
    case (TheHole): {
      accumulator = value;
    }
    case (accumulatorNotHole: JSAny): {
      // 这里调用 callbackfn，最后面的 4 个参数
      // 传给 callbackfn 
      accumulator = Call(
          context, callbackfn, Undefined, accumulatorNotHole, value, k,
          fastOW.Get());
    }
  }
}
return accumulator;
```

## 循环次数在 for 循环前确定

如果在 filter 接收的 callback 回调函数中，改变数组的长度，并不影响 for 循环次数。比如下面的代码：

```Javascript
const numArray = [1, 2, 3]
const sum  = numArray.filter((acc, cur) => {
  if (cur === 1) {
    numArray.push(4, 5, 6)
  }
  console.log('tick')
  return acc + cur
}, 0)
console.log(sum) // 打印 6
console.log(numArray) // 打印 [1, 2, 3, 4, 5, 6]
```

在调用 filter 之前，numArray 的长度是 3，尽管在调用 filter 过程中 numArray 长度变为 6，但循环只发生 3 次，打印 3 个 tick，sum 结果为 6。

![filter3](https://raw.githubusercontent.com/xudale/blog/master/assets/filter3.png)

ecma 规范中有提及循环次数在 for 循环前确定：

> The range of elements processed by filter is set before the first call to callbackfn. Elements that are appended to the array after the call to filter begins will not be visited by callbackfn. If existing elements of the array are changed, their value as passed to callbackfn will be the value at the time filter visits them; elements that are deleted after the call to filter begins and before being visited are not visited.

## 若在循环过程中改变了数组元素，则遍历到的是改变后的元素

如果在 filter 接收的 callback 回调函数中，改变了数组元素，则后面的循环遍历到的是改变后的元素，比如下面代码：

```Javascript
const numArray = [1, 2, 3]
const sum  = numArray.filter((acc, cur) => {
  if (cur === 1) {
    numArray[2] = 9999999999
  }
  console.log('tick')
  return acc + cur
}, 0)
console.log(sum) // 打印 10000000002
console.log(numArray) // 打印 [1, 2, 9999999999]
```

在遍历过程中，通过 numArray[2] = 9999999999，改变了数组元素，当遍历到数组最后一个元素时，此时的 cur 是 9999999999，不再是数组初始化时的 3。

![filter9](https://raw.githubusercontent.com/xudale/blog/master/assets/filter9.png)

相关规范如下：

![filterEcma](https://raw.githubusercontent.com/xudale/blog/master/assets/filterEcma.png)

在日常业务开发中，每年差不多都能遇到一次希望在 forEach/filter 里删除/改变元素的场景。比如一个文本框列表，点提交时，既要做用户的输入内容校验，又要从提交的数据中删除未填写的文本框。这种场景可维护性高而又优雅的写法当然是：

```Javascript
array.filter(someFunction1).forEach/filter(someFunction2)
```

先调用 filter 删除未填写的文本框，然后 forEach/filter 做校验或拼装提交给后端的数据。但遇到大数组的情况，从性能考虑，还是希望一次遍历解决问题。只要了解 filter 相关的规范和源码，在这种极端情况下，可以考虑在一次 forEach/filter 里面处理。


## 简易版 filter

如果不考虑任何边界条件，尽可能模仿 V8 FastArrayfilter 的实现逻辑，filter 方法可用 Javascript 实现如下：

```Javascript
function filter(...args) {
  const array = this
  const callBack = args[0]
  let accumulator = args[1] || undefined
  const len = array.length
  for (let i = 0; i < len; i++) {
    if (accumulator) {
      accumulator = callBack(accumulator, array[i], i, array)
    } else {
      accumulator = array[i]
    }
  }
  return accumulator
}
```

上面代码最明显的一个 bug 是如果 i > 0，但 callback 返回了 null/undefined，下一次循环则直接进入了 else 分支 accumulator = array[i]。为了简洁，笔者不改了，mdn 有符合规范的 filter Javascript 实现版本，见参考文献。


## 参考文献

[ecma262:sec-array.prototype.filter](https://tc39.es/ecma262/#sec-array.prototype.filter)

[mdn:Array.prototype.filter](https://developer.mozilla.org/zh-TW/docs/Web/JavaScript/Reference/Global_Objects/Array/filter)


















