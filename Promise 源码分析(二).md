# Promise 源码分析(二)

基于 node 版本 14.13.0，V8 版本 8.4.371。本文介绍的内容是 reject、catch 和 then 的链式调用。

## reject

```JavaScript
new Promise((resolve, reject) => {
  setTimeout(_ => reject('rejected'), 5000)
}).then(_ => {
  console.log('fulfilled')
}, reason => {
  console.log(reason)
})
```

上述代码 5s 后执行了 reject 函数，控制台打印 rejected。reject 函数调用了 V8 的 [RejectPromise](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-abstract-operations.tq) 函数，源码如下：

```C++
transitioning builtin
RejectPromise(implicit context: Context)(
    promise: JSPromise, reason: JSAny, debugEvent: Boolean): JSAny {
  // 取出 Promise 的处理对象 PromiseReaction
  const reactions =
      UnsafeCast<(Zero | PromiseReaction)>(promise.reactions_or_result);
  // 这里的 reason 就是 reject 函数的参数
  promise.reactions_or_result = reason;
  // 设置 Promise 的状态为 rejected
  promise.SetStatus(PromiseState::kRejected);
  TriggerPromiseReactions(reactions, reason, kPromiseReactionReject);
  return Undefined;
}
```

[TriggerPromiseReactions](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-abstract-operations.tq#140) 函数在上一篇文章分析过，功能是将 Promise 的处理对象 PromiseReaction 链表，依次插入 V8 的 microtask 队列，源码继续删减如下：

```C++
// https://tc39.es/ecma262/#sec-triggerpromisereactions
transitioning macro TriggerPromiseReactions(implicit context: Context)(
    reactions: Zero|PromiseReaction, argument: JSAny,
    reactionType: constexpr PromiseReactionType): void {
  // 删减了链表反转的代码
  let current = reactions;
  // reactions 是一个链表，下面的 while 循环遍历链表
  while (true) {
    typeswitch (current) {
      case (Zero): {
        break;
      }
      case (currentReaction: PromiseReaction): {
        // 取出链表下一个结点
        current = currentReaction.next; 
        // 调用 MorphAndEnqueuePromiseReaction，把链接中的每一项都进入 microtask 队列
        MorphAndEnqueuePromiseReaction(currentReaction, argument, reactionType);
      }
    }
  }
}
```

[MorphAndEnqueuePromiseReaction](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-abstract-operations.tq#84) 将 PromiseReaction 转为 microtask，最终放入 microtask 队列，morph 本身有转变/转化的意思，比如多态的英文是 Polymorphism。MorphAndEnqueuePromiseReaction 接收 3 个参数，PromiseReaction 就是前面提到的包装了 Promise 处理函数的对象，argument 与 Promise 最后的状态，可能是 Promise 的 value/reason，reactionType 表示 Promise 最终的状态，fulfilled 状态对应的值是 kPromiseReactionFulfill，rejected 状态对应的值是 kPromiseReactionReject。MorphAndEnqueuePromiseReaction 的逻辑很简单，因为此时已经知道了 Promise 的最终状态，所以可以从 promiseReaction 对象得到 promiseReactionJobTask 对象，promiseReactionJobTask 的变量命名与 ECMA 规范相关描述一脉相承，其实就是 microtask。MorphAndEnqueuePromiseReaction 源码如下，仅保留了和本小节相关的内容。

```C++
transitioning macro MorphAndEnqueuePromiseReaction(implicit context: Context)(
    promiseReaction: PromiseReaction, argument: JSAny,
    reactionType: constexpr PromiseReactionType): void {
  let primaryHandler: Callable|Undefined;
  let secondaryHandler: Callable|Undefined;
  if constexpr (reactionType == kPromiseReactionFulfill) {
    primaryHandler = promiseReaction.fulfill_handler;
    secondaryHandler = promiseReaction.reject_handler;
  } else {
    primaryHandler = promiseReaction.reject_handler;
    secondaryHandler = promiseReaction.fulfill_handler;
  }
  const handlerContext: Context =
      ExtractHandlerContext(primaryHandler, secondaryHandler);
  if constexpr (reactionType == kPromiseReactionFulfill) {
    // 删
  } else {
    StaticAssert(reactionType == kPromiseReactionReject);
    * UnsafeConstCast(& promiseReaction.map) =
        PromiseRejectReactionJobTaskMapConstant();
    const promiseReactionJobTask =
        UnsafeCast<PromiseRejectReactionJobTask>(promiseReaction);
    // argument 是 reject 的参数
    promiseReactionJobTask.argument = argument;
    promiseReactionJobTask.context = handlerContext;
    // handler 是 JS 层面 then 方法的第二个参数，或 catch 方法的参数
    promiseReactionJobTask.handler = primaryHandler;
    // promiseReactionJobTask 就是那个工作中经常被反复提起的 microtask
    // EnqueueMicrotask 将 microtask 插入 microtask 队列
    EnqueueMicrotask(handlerContext, promiseReactionJobTask);
    StaticAssert(
        kPromiseReactionPromiseOrCapabilityOffset ==
        kPromiseReactionJobTaskPromiseOrCapabilityOffset);
  }
}
```

reject 和 resolve 的逻辑基本相同，分 3 步：
- 设置 Promise 的 value/reason，也就是 resolve/reject 的参数
- 设置 Promise 的状态：fulfilled/rejected
- 从之前调用 then 方法时收集到的依赖，也就是 promiseReaction 对象，得到 microtask，最后将 microtask 插入 microtask 队列
## catch

```JavaScript
new Promise((resolve, reject) => {
    setTimeout(reject, 2000)
}).catch(_ => {
    console.log('rejected')
})
```

当 catch 方法执行时，调用了 V8 的 [PromisePrototypeCatch](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-constructor.tq#100) 方法，源码如下：

```C++
transitioning javascript builtin
PromisePrototypeCatch(
    js-implicit context: Context, receiver: JSAny)(onRejected: JSAny): JSAny {
  const nativeContext = LoadNativeContext(context);
  return UnsafeCast<JSAny>(
      InvokeThen(nativeContext, receiver, Undefined, onRejected));
}
```

PromisePrototypeCatch 的源码确实就这几行，调用了 InvokeThen 方法。从名字可以推测出，InvokeThen 调用的就是 then 方法，[InvokeThen](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-misc.tq#199) 源码如下：

```C++
transitioning
macro InvokeThen<F: type>(implicit context: Context)(
    nativeContext: NativeContext, receiver: JSAny, arg1: JSAny, arg2: JSAny,
    callFunctor: F): JSAny {
  if (!Is<Smi>(receiver) &&
      IsPromiseThenLookupChainIntact(
          nativeContext, UnsafeCast<HeapObject>(receiver).map)) {
    const then =
        UnsafeCast<JSAny>(nativeContext[NativeContextSlot::PROMISE_THEN_INDEX]);
    return callFunctor.Call(nativeContext, then, receiver, arg1, arg2);
  } else
    deferred {
      const then = UnsafeCast<JSAny>(GetProperty(receiver, kThenString));
      return callFunctor.Call(nativeContext, then, receiver, arg1, arg2);
    }
}
```

InvokeThen 方法有 if/else 两个分支，两个分支的逻辑差不多，本文的 JS 示例代码走的是 if 分支。先是拿到 V8 原生的 then 方法，然后通过 callFunctor.Call(nativeContext, then, receiver, arg1, arg2) 调用 then 方法。then 方法上一篇文章分享过，这里不再赘述。

既然 catch 方法底层调用了 then 方法，那么 catch 方法也有和 then 方法一样的返回值。catch 方法可以继续抛出异常，可以继续链式调用。

```JavaScript
new Promise((resolve, reject) => {
    setTimeout(reject, 2000)
}).catch(_ => {
    throw 'rejected'
}).catch(_ => {
    console.log('last catch')
})
```

上面的代码第 2 个 catch 处理第 1 个 catch 抛出的异常，最后打印 last catch。

> catch 方法底层调用的是 then 方法
> 
> JS 层面 obj.catch(onRejected) 等价于 obj.then(undefined, onRejected)

## then 的链式调用与 microtask 队列

```JavaScript
Promise.resolve('123')
    .then(() => {throw new Error('456')})
    .then(_ => {
        console.log('shouldnot be here')
    })
    .catch((e) => console.log(e))
    .then((data) => console.log(data));
```

以上代码运行后，打印 Error: 456 和 undefined。为了便于叙述，将 then 的链式调用写法改为啰嗦写法。

```JavaScript
const p0 = Promise.resolve('123')
const p1 = p0.then(() => {throw new Error('456')})
const p2 = p1.then(_ => {
    console.log('shouldnot be here')
})
const p3 = p2.catch((e) => console.log(e))
const p4 = p3.then((data) => console.log(data));
```

then 方法返回新的 Promise，所以 p0、p1、p2、p3 和 p4 这 5 个 Promise 互不相等。

p0 开始便处于 fulfilled 状态，当执行

```JavaScript
const p1 = p0.then(() => {throw new Error('456')})
```

时，由于 p0 已是 fulfilled 状态，直接将 p0 的 fulfilled 处理函数插入 microtask 队列，此时 microtask 简略示意图如下：



p1 最终是 rejected 状态，但 p1 只有 fulfilled 状态的处理函数，没有 rejected 状态的处理函数

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

## 总结与感想

曾经觉得 Promise 很神秘，看了源码觉得 Promise 的本质其实还是回调函数，只不过背靠 Promise 的一系列方法和思想，改变了书写回调函数的方式。then 方法做依赖收集。resolve 收集到的依赖，放入 microtask 队列中。Promise 属于微创新，async/await 抛弃回调函数式的写法，暂停/恢复当前代码的执行，是革命性的创新。

![promiseConclude](https://raw.githubusercontent.com/xudale/blog/master/assets/promiseConclude.png)












