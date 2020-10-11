# Promise 源码分析(一)

本文基于 node 版本 14.13.0，V8 版本 8.4.371。Promise 源码全部在 V8，node 版本不重要。很早就想写 Promise 源码相关的文章，由于 V8 中 Promise 的源码之前（18、19年）是用 CodeStubAssembler 写的，很多前端看不懂 CodeStubAssembler 这种汇编风格的语言，考虑现实选择放弃。直到两个月前无意中发现 V8 的新版本已经用 Torque 重写了 Promise 的大部分代码，遂有此文。预计写 2-3 篇文章，本文介绍的内容是 Promise 构造函数、then 和 resolve。

## 基本数据结构

### 三种状态

Promise 共有 3 种状态，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/base.tq#190)：

```C++
// Promise constants
extern enum PromiseState extends int31 constexpr 'Promise::PromiseState' {
  kPending,
  kFulfilled,
  kRejected
}
```

一个新创建的 Promise 处于 pending 状态。当调用 resolve 或 reject 函数后，Promise 处于 fulfilled 或 rejected 状态，此后 Promise 的状态保持不变，也就是说 Promise 的状态改变是不可逆的，Promise 源码中出现了多处状态相关 assert。

### JSPromise

JSPromise 描述 Promise 的基本信息，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/objects/js-promise.tq#13)：

```C++
bitfield struct JSPromiseFlags extends uint31 {
  status: PromiseState: 2 bit; // Promise 的状态，kPending/kFulfilled/kRejected
  has_handler: bool: 1 bit; // 是否有处理函数，没有调用过 then 方法的 Promise 没有处理函数
  handled_hint: bool: 1 bit;
  async_task_id: int32: 22 bit;
}

@generateCppClass
extern class JSPromise extends JSObject {
  macro Status(): PromiseState {
    // 获取 Promise 的状态，返回 kPending/kFulfilled/kRejected 中的一个
    return this.flags.status;
  }

  macro SetStatus(status: constexpr PromiseState): void {
    // 第 1 个 assert 表示只有 pending 状态的 Promise 才可以被改变状态
    assert(this.Status() == PromiseState::kPending);
    // 第 2 个 assert 表示 Promise 创建成功后，不可将 Promise 设置为 pending 状态
    assert(status != PromiseState::kPending);
    this.flags.status = status;
  }

  macro HasHandler(): bool {
    // 判断 Promise 是否有处理函数
    return this.flags.has_handler;
  }

  macro SetHasHandler(): void {
    this.flags.has_handler = true;
  }

  // Smi 0 terminated list of PromiseReaction objects in case the JSPromise was
  // not settled yet, otherwise the result.
  // promise 处理函数或结果，可以是无/包装了 onFulfilled/onRejected 回调函数的对象/resolve 接收的参数
  reactions_or_result: Zero|PromiseReaction|JSAny;
  flags: SmiTagged<JSPromiseFlags>;
}
```

当 Promise 状态改变时，比如调用了 resolve/reject 函数，SetStatus 方法会被调用；Javascript 层调用 resolve 方法时，reactions_or_result 字段会被赋值；Javascript 层调用 then 方法时，说明已经有了处理函数，SetHasHandler 会被调用。Status/SetStatus 这两个方法一个获取 Promise 状态，一个设置 Promise 状态，因为容易理解，所以不再举例；下面举一个 HasHandler/SetHasHandler 的例子。

```JavaScript
const myPromise1 = new Promise((resolve, reject) => {
  reject()
})
```

在 node-v14.13.0 环境下执行，结果如下：

![unhandleReject](https://raw.githubusercontent.com/xudale/blog/master/assets/unhandleReject.png)

大意是说处于 rejected 状态的 Promise 必须要有处理函数。V8 通过 HasHandler 判断 myPromise1 并没有处理函数。当把处理函数加上以后，代码如下：

```JavaScript
const myPromise1 = new Promise((resolve, reject) => {
  reject()
})

myPromise1.then(console.log, console.log)
```

此时 SetHasHandler 已被调用，HasHandler 返回 true 表示 myPromise1 有处理函数。在 node-v14.13.0 环境下执行，没有错误提示，一切正常。

### 其它

- executor：是函数，是 Promise 构造函数接收的参数
- PromiseReaction：是对象，表示 Promise 的处理函数，因为一个 Promise 多次调用 then 方法就会有多个处理函数，所以底层数据结构是个链表。

## 构造函数

构造函数[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-constructor.tq#47)：

```C++
// https://tc39.es/ecma262/#sec-promise-executor
transitioning javascript builtin
PromiseConstructor(
    js-implicit context: NativeContext, receiver: JSAny,
    newTarget: JSAny)(executor: JSAny): JSAny {
  // 1. If NewTarget is undefined, throw a TypeError exception.
  if (newTarget == Undefined) {
    ThrowTypeError(MessageTemplate::kNotAPromise, newTarget);
  }

  // 2. If IsCallable(executor) is false, throw a TypeError exception.
  if (!Is<Callable>(executor)) {
    ThrowTypeError(MessageTemplate::kResolverNotAFunction, executor);
  }

  let result: JSPromise;
  // 构造一个 Promise 对象
  result = NewJSPromise();
  // 从 Promise 对象 result 身上，获取它的 resolve 和 reject 函数
  const funcs = CreatePromiseResolvingFunctions(result, True, context);
  const resolve = funcs.resolve;
  const reject = funcs.reject;
  try {
    // 直接同步调用 executor 函数，resolve 和 reject 做为参数
    Call(context, UnsafeCast<Callable>(executor), Undefined, resolve, reject);
  } catch (e) {
    Call(context, reject, Undefined, e);
  }
  return result;
}
```

首先分析两个 ThrowTypeError，以下代码可触发第一个 ThrowTypeError。

```JavaScript
Promise()  // Uncaught TypeError: undefined is not a promise
```

原因是没有使用 new 操作符调用 Promise 构造函数，此时 newTarget 等于 Undefined，触发了 ThrowTypeError(MessageTemplate::kNotAPromise, newTarget)。

以下代码可触发第二个 ThrowTypeError。

```JavaScript
new Promise() // Uncaught TypeError: Promise resolver undefined is not a function
```

此时 newTarget 不等于 Undefined，不会触发第一个 ThrowTypeError。但调用 Promise 构造函数时没传参数 executor，触发了第二个 ThrowTypeError。

错误消息在 C++ 代码中定义，使用了宏和枚举巧妙的生成了 C++ 代码，这里不做展开，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/common/message-template.h#130)：

```C++
T(NotAPromise, "% is not a promise")                            \
T(ResolverNotAFunction, "Promise resolver % is not a function") \
```

executor 的类型是函数，在 JavaScript 的世界里，回调函数通常是异步调用，但 executor 是同步调用。在 Call(context, UnsafeCast<Callable>(executor), Undefined, resolve, reject) 这一行，同步调用了 executor。

```JavaScript
console.log('同步执行开始')
new Promise((resolve, reject) => {
  resolve()
  console.log('executor 同步执行')
})

console.log('同步执行结束')
// 本段代码的打印顺序是:
// 同步执行开始
// executor 同步执行
// 同步执行结束
```

> Promise 构造函数接收的参数 executor，是被同步调用的

Promise 构造函数调用 NewJSPromise 获取一个新的 JSPromise 对象。NewJSPromise 调用 [PromiseInit](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-misc.tq#29) 来初始化一个 JSPromise 对象，源码如下：

```C++
macro PromiseInit(promise: JSPromise): void {
  promise.reactions_or_result = kZero;
  promise.flags = SmiTag(JSPromiseFlags{
    status: PromiseState::kPending,
    has_handler: false,
    handled_hint: false,
    async_task_id: 0
  });
  promise_internal::ZeroOutEmbedderOffsets(promise);
}
```

PromiseInit 函数初始化 JSPromise 对象，相关字段可与本文开头介绍的 JSPromise 对象互相印证。


## then

### PromisePrototypeThen
JavaScript 层的 then 函数实际上是 V8 中的 PromisePrototypeThen 函数，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-then.tq#21)：

```C++
transitioning javascript builtin
PromisePrototypeThen(js-implicit context: NativeContext, receiver: JSAny)(
    onFulfilled: JSAny, onRejected: JSAny): JSAny {
  // 1. Let promise be the this value.
  // 2. If IsPromise(promise) is false, throw a TypeError exception.
  const promise = Cast<JSPromise>(receiver) otherwise ThrowTypeError(
      MessageTemplate::kIncompatibleMethodReceiver, 'Promise.prototype.then',
      receiver);

  // 3. Let C be ? SpeciesConstructor(promise, %Promise%).
  const promiseFun = UnsafeCast<JSFunction>(
      context[NativeContextSlot::PROMISE_FUNCTION_INDEX]);

  // 4. Let resultCapability be ? NewPromiseCapability(C).
  let resultPromiseOrCapability: JSPromise|PromiseCapability;
  let resultPromise: JSAny;
  label AllocateAndInit {
    const resultJSPromise = NewJSPromise(promise);
    resultPromiseOrCapability = resultJSPromise;
    resultPromise = resultJSPromise;
  }
  // onFulfilled 和 onRejected 是 then 接收的两个参数
  const onFulfilled = CastOrDefault<Callable>(onFulfilled, Undefined);
  const onRejected = CastOrDefault<Callable>(onRejected, Undefined);

  // 5. Return PerformPromiseThen(promise, onFulfilled, onRejected,
  //    resultCapability).
  PerformPromiseThenImpl(
      promise, onFulfilled, onRejected, resultPromiseOrCapability);
  // 返回一个新的 Promise
  return resultPromise;
}
```

PromisePrototypeThen 函数创建了一个新的 Promise，获取 then 接收到的两个参数，调用 PerformPromiseThenImpl。这里有一点值得注意，then 方法返回的是一个新创建的 Promise。

```JavaScript
const myPromise2 = new Promise((resolve, reject) => {
  resolve('foo')
})

const myPromise3 = myPromise2.then(console.log)

// myPromise2 和 myPromise3 是两个不同的对象，可以有不同的状态
console.log(myPromise2 === myPromise3) // 打印 false
```

### PerformPromiseThenImpl

PerformPromiseThenImpl [源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-abstract-operations.tq#409)：

```C++
transitioning macro PerformPromiseThenImpl(implicit context: Context)(
    promise: JSPromise, onFulfilled: Callable|Undefined,
    onRejected: Callable|Undefined,
    resultPromiseOrCapability: JSPromise|PromiseCapability|Undefined): void {
  if (promise.Status() == PromiseState::kPending) {
    // penging 状态的分支
    // The {promise} is still in "Pending" state, so we just record a new
    // PromiseReaction holding both the onFulfilled and onRejected callbacks.
    // Once the {promise} is resolved we decide on the concrete handler to
    // push onto the microtask queue.
    const handlerContext = ExtractHandlerContext(onFulfilled, onRejected);
    // 拿到 Promise 的 reactions_or_result 字段
    const promiseReactions =
        UnsafeCast<(Zero | PromiseReaction)>(promise.reactions_or_result);
    // 考虑一个 Promise 可能会有多个 then 的情况，reaction 是个链表
    // 存 Promise 的所有处理函数
    const reaction = NewPromiseReaction(
        handlerContext, promiseReactions, resultPromiseOrCapability,
        onFulfilled, onRejected);
    promise.reactions_or_result = reaction;
  } else {
    // fulfilled 和 rejected 状态的分支
    const reactionsOrResult = promise.reactions_or_result;
    let microtask: PromiseReactionJobTask;
    let handlerContext: Context;
    if (promise.Status() == PromiseState::kFulfilled) {
      handlerContext = ExtractHandlerContext(onFulfilled, onRejected);
      microtask = NewPromiseFulfillReactionJobTask(
          handlerContext, reactionsOrResult, onFulfilled,
          resultPromiseOrCapability);
    } else
      deferred {
        assert(promise.Status() == PromiseState::kRejected);
        handlerContext = ExtractHandlerContext(onRejected, onFulfilled);
        microtask = NewPromiseRejectReactionJobTask(
            handlerContext, reactionsOrResult, onRejected,
            resultPromiseOrCapability);
        if (!promise.HasHandler()) {
          runtime::PromiseRevokeReject(promise);
        }
      }
    // 即使调用 then 方法时 promise 已经 fulfilled 或 rejected
    // then 方法的 onFulfilled 或 onRejected 参数也不会立刻执行
    // 而是进入 microtask 队列后执行
    EnqueueMicrotask(handlerContext, microtask);
  }
  promise.SetHasHandler();
}
```

### PerformPromiseThenImpl penging 分支

PerformPromiseThenImpl 有两个分支，pending 分支调用 NewPromiseReaction 函数，在接收到的 onFulfilled 和 onRejected 参数的基础上，生成 PromiseReaction 对象，并赋值给 JSPromise 的 reactions_or_result 字段;

考虑一个 Promise 可以会连续调用多个 then 的情况，比如：

```JavaScript
const myPromise4 = new Promise((resolve, reject) => {
  setTimeout(_ => {
    resolve('my code delay 5000') 
  }, 5e3)
})

myPromise4.then(result => {
  console.log('第 1 个 then')
})

myPromise4.then(result => {
  console.log('第 2 个 then')
})
```

myPromise4 调用了两次 then 方法，每个 then 方法都会生成一个 PromiseReaction 对象。第一次调用 then 方法时生成对象 PromiseReaction1，此时 myPromise4 的 reactions_or_result 存的是 PromiseReaction1。

第二次调用 then 方法时生成对象 PromiseReaction2，调用 NewPromiseReaction 函数时，PromiseReaction2.next = PromiseReaction1，PromiseReaction1 变成了 PromiseReaction2 的下一个节点，最后 myPromise4 的 reactions_or_result 存的是 PromiseReaction2。NewPromiseReaction 函数[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-misc.tq#134)：

```C++
macro NewPromiseReaction(implicit context: Context)(
    handlerContext: Context, next: Zero|PromiseReaction,
    promiseOrCapability: JSPromise|PromiseCapability|Undefined,
    fulfillHandler: Callable|Undefined,
    rejectHandler: Callable|Undefined): PromiseReaction {
  const nativeContext = LoadNativeContext(handlerContext);
  return new PromiseReaction{
    map: PromiseReactionMapConstant(),
    next: next, // next 字段存的是链表中的下一个节点
    reject_handler: rejectHandler,
    fulfill_handler: fulfillHandler,
    promise_or_capability: promiseOrCapability,
    continuation_preserved_embedder_data: nativeContext
        [NativeContextSlot::CONTINUATION_PRESERVED_EMBEDDER_DATA_INDEX]
  };
}
```

在 myPromise4 处于 pending 状态时，myPromise4 的 reactions_or_result 字段如下图： 

![reactionList](https://raw.githubusercontent.com/xudale/blog/master/assets/reactionList.png)

### PerformPromiseThenImpl fulfilled/rejected 分支

fulfilled/rejected 分支逻辑则简单的多，处理的是当 Promise 处于 fulfilled/rejected 状态时，调用 then 方法的逻辑，以 fulfilled 状态为例，调用 NewPromiseFulfillReactionJobTask 生成 microtask，然后 EnqueueMicrotask(handlerContext, microtask) 将刚才生成的 microtask 放入 microtask 队列。

```JavaScript
new Promise((resolve, reject) => {
  resolve()
}).then(result => {
  console.log('进入 microtask 队列后执行')
})

console.log('同步执行结束')
// 打印顺序：同步执行结束、进入 microtask 队列后执行
```

尽管调用 then 方法时，Promise 已经处于 resolved 状态，但 then 方法的 fulfill 回调函数不会立刻执行，而是进入了 microtask 队列后执行。

> 对于一个已经 fulfilled 状态的 Promise，调用其 then 方法，then 方法接收的 fulfill 回调函数不会立即执行。而是进入 microtask 队列后执行。

## resolve

当调用 resolve 函数时，归要到底调用了 V8 的 FulfillPromise 函数，[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-abstract-operations.tq#182)：

```C++
// https://tc39.es/ecma262/#sec-fulfillpromise
transitioning builtin
FulfillPromise(implicit context: Context)(
    promise: JSPromise, value: JSAny): Undefined {
  // Assert: The value of promise.[[PromiseState]] is "pending".
  // promise 的状态改变是不可逆的
  assert(promise.Status() == PromiseState::kPending);

  // 2. Let reactions be promise.[[PromiseFulfillReactions]].
  const reactions =
      UnsafeCast<(Zero | PromiseReaction)>(promise.reactions_or_result);

  // 3. Set promise.[[PromiseResult]] to value.
  // 4. Set promise.[[PromiseFulfillReactions]] to undefined.
  // 5. Set promise.[[PromiseRejectReactions]] to undefined.
  promise.reactions_or_result = value;

  // 6. Set promise.[[PromiseState]] to "fulfilled".
  promise.SetStatus(PromiseState::kFulfilled);

  // 7. Return TriggerPromiseReactions(reactions, value).
  TriggerPromiseReactions(reactions, value, kPromiseReactionFulfill);
  return Undefined;
}
```

获取 Promise 的处理函数到 reactions，reactions 的类型是 PromiseReaction，是个链表，上节有讲过；设置 promise 的 reactions_or_result 为 value，这个 value 就是 JavaScript 层传给 resolve 的参数，调用 promise.SetStatus 设置 promise 的状态为 PromiseState::kFulfilled，最后调用 TriggerPromiseReactions。[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-abstract-operations.tq#140)：

```C++
// https://tc39.es/ecma262/#sec-triggerpromisereactions
transitioning macro TriggerPromiseReactions(implicit context: Context)(
    reactions: Zero|PromiseReaction, argument: JSAny,
    reactionType: constexpr PromiseReactionType): void {
  // We need to reverse the {reactions} here, since we record them on the
  // JSPromise in the reverse order.
  let current = reactions;
  let reversed: Zero|PromiseReaction = kZero;
  // 链表反转
  while (true) {
    typeswitch (current) {
      case (Zero): {
        break;
      }
      case (currentReaction: PromiseReaction): {
        current = currentReaction.next;
        currentReaction.next = reversed;
        reversed = currentReaction;
      }
    }
  }
  // Morph the {reactions} into PromiseReactionJobTasks and push them
  // onto the microtask queue.
  current = reversed;
  // 链表反转后，调用 MorphAndEnqueuePromiseReaction，把链接中的每一项都进入 microtask 队列
  while (true) {
    typeswitch (current) {
      case (Zero): {
        break;
      }
      case (currentReaction: PromiseReaction): {
        current = currentReaction.next;
        MorphAndEnqueuePromiseReaction(currentReaction, argument, reactionType);
      }
    }
  }
}
```

TriggerPromiseReactions 做了两件事：

- 反转 reactions 链表，前文有分析过 then 方法的实现，then 方法的参数最存在链表中，并且最后被调用的 then 方法，它接收的参数会位于链表的头部，这不符合规范，所以需要反转
- 遍历 reactions 对象，将每个元素放入 microtask 队列

```JavaScript
const myPromise5 = new Promise((resolve, reject) => {
  setTimeout(_ => {
    resolve('my code delay 5000') 
  }, 5e3)
})

myPromise5.then(result => {
  console.log('第 1 个 then')
})

myPromise5.then(result => {
  console.log('第 2 个 then')
})
// 打印顺序：第 1 个 then、第 2 个 then
```

笔者把 TriggerPromiseReactions 中链表反转的代码注释掉，上面代码的打印顺序则变成了第 2 个 then、第 1 个 then。

## 总结与感想

曾经觉得 Promise 很神秘，看了源码觉得 Promise 的本质其实还是回调函数，只不过靠着 Promise 的一系列方法和思想，改变了回调函数的出现的位置。then 方法一路看下来，做的是依赖收集的工作。resolve 将 then 方法收集到的依赖，放入 microtask 队列中。

![promiseConclude](https://raw.githubusercontent.com/xudale/blog/master/assets/promiseConclude.png)












