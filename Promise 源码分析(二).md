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

上述代码 5s 后执行了 reject 函数，控制台打印 rejected。reject 函数调用了 V8 的 [RejectPromise](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-abstract-operations.tq#210) 函数，源码如下：

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

[TriggerPromiseReactions](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-abstract-operations.tq#140) 函数在上一篇文章分析过，功能是将 Promise 处理函数相关的 PromiseReaction 链表，反转后依次插入 V8 的 microtask 队列，TriggerPromiseReactions 源码继续删减如下：

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
        // 调用 MorphAndEnqueuePromiseReaction，将当前节点插入 microtask 队列
        MorphAndEnqueuePromiseReaction(currentReaction, argument, reactionType);
      }
    }
  }
}
```

[MorphAndEnqueuePromiseReaction](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-abstract-operations.tq#84) 将 PromiseReaction 转为 microtask，最终放入 microtask 队列，morph 本身有转变/转化的意思，比如 Polymorphism (多态)。MorphAndEnqueuePromiseReaction 接收 3 个参数，PromiseReaction 是前面提到的包装了 Promise 处理函数的链表对象，argument 是 resolve/reject 的参数，reactionType 表示 Promise 最终的状态，fulfilled 状态对应的值是 kPromiseReactionFulfill，rejected 状态对应的值是 kPromiseReactionReject。MorphAndEnqueuePromiseReaction 的逻辑很简单，因为此时已经知道了 Promise 的最终状态，所以可以从 promiseReaction 对象得到 promiseReactionJobTask 对象，promiseReactionJobTask 的变量命名与 ECMA 规范相关描述一脉相承，其实就是传说中的 microtask。MorphAndEnqueuePromiseReaction 源码如下，仅保留了和本小节相关的内容。

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
  }
}
```

reject 和 resolve 的逻辑基本相同，分为 3 步：
- 设置 Promise 的 value/reason，也就是 resolve/reject 的参数
- 设置 Promise 的状态：fulfilled/rejected
- 从之前调用 then/catch 方法时收集到的依赖，也就是 promiseReaction 对象，得到一个个 microtask，最后将 microtask 插入 microtask 队列

## catch

```JavaScript
new Promise((resolve, reject) => {
    setTimeout(reject, 2000)
}).catch(_ => {
    console.log('rejected')
})
```

以上面代码为例，当 catch 方法执行时，调用了 V8 的 [PromisePrototypeCatch](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-constructor.tq#100) 方法，源码如下：

```C++
transitioning javascript builtin
PromisePrototypeCatch(
    js-implicit context: Context, receiver: JSAny)(onRejected: JSAny): JSAny {
  const nativeContext = LoadNativeContext(context);
  return UnsafeCast<JSAny>(
      InvokeThen(nativeContext, receiver, Undefined, onRejected));
}
```

PromisePrototypeCatch 的源码确实就这几行，除了调用 InvokeThen 方法再无其它
。从名字可以推测出，InvokeThen 调用的是 Promise 的 then 方法，[InvokeThen](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-misc.tq#199) 源码如下：

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
    // 重点在下面一行，调用 then 方法并返回，两个分支都一样
    return callFunctor.Call(nativeContext, then, receiver, arg1, arg2);
  } else
    deferred {
      const then = UnsafeCast<JSAny>(GetProperty(receiver, kThenString));
      // 重点在下面一行，调用 then 方法并返回，两个分支都一样
      return callFunctor.Call(nativeContext, then, receiver, arg1, arg2);
    }
}
```

InvokeThen 方法有 if/else 两个分支，两个分支的逻辑差不多，本小节的 JS 示例代码走的是 if 分支。先是拿到 V8 原生的 then 方法，然后通过 callFunctor.Call(nativeContext, then, receiver, arg1, arg2) 调用 then 方法。then 方法上一篇文章有提及，这里不再赘述。

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

上面的代码第 2 个 catch 捕获第 1 个 catch 抛出的异常，最后打印 last catch。

> catch 方法通过底层调用 then 方法来实现
> 
> 假如 obj 是一个 Promise 对象，JS 层面 obj.catch(onRejected) 等价于 obj.then(undefined, onRejected)

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

时，由于 p0 已是 fulfilled 状态，直接将 p0 的 fulfilled 处理函数插入 microtask 队列，此时 microtask 队列简略示意图如下，绿色区域表示 microtask，蓝色区域表示 microtask 队列。

![promise1](https://raw.githubusercontent.com/xudale/blog/master/assets/p1.png)

跑完余下所有的代码。

```JavaScript
const p1 = p0.then(() => {throw new Error('456')})
const p2 = p1.then(_ => {
    console.log('shouldnot be here')
})
const p3 = p2.catch((e) => console.log(e))
const p4 = p3.then((data) => console.log(data));
```

p1、p2、p3 和 p4 这 4 个 Promise 都处于 pending 状态，microtask 队列还是

![promise1](https://raw.githubusercontent.com/xudale/blog/master/assets/p1.png)

开始执行 microtask 队列，核心方法是 [MicrotaskQueueBuiltinsAssembler::RunSingleMicrotask](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/builtins-microtask-queue-gen.cc#114)，代码是用 CodeStubAssembler 写的，代码很长，逻辑简单，评论区经常有提看不懂 CodeStubAssembler 这种类汇编语言，这里就不再贴代码了，预计之后的版本 V8 会用 Torque 重写的。

在执行 microtask 的过程中，MicrotaskQueueBuiltinsAssembler::RunSingleMicrotask 会调用 [PromiseReactionJob](https://chromium.googlesource.com/v8/v8.git/+/refs/heads/8.4-lkgr/src/builtins/promise-reaction-job.tq#73)，源码如下：

```C++
transitioning
macro PromiseReactionJob(
    context: Context, argument: JSAny, handler: Callable|Undefined,
    promiseOrCapability: JSPromise|PromiseCapability|Undefined,
    reactionType: constexpr PromiseReactionType): JSAny {
  if (handler == Undefined) {
    // 没有处理函数的 case，Promise 的状态相当于透传
    if constexpr (reactionType == kPromiseReactionFulfill) {
      // 基本类同 JS 层的 resolve
      return FuflfillPromiseReactionJob(
          context, promiseOrCapability, argument, reactionType);
    } else {
      // 基本类同 JS 层的 reject
      return RejectPromiseReactionJob(
          context, promiseOrCapability, argument, reactionType);
    }
  } else {
    try {
      // 试图调用 Promise 处理函数
      const result =
          Call(context, UnsafeCast<Callable>(handler), Undefined, argument);
        // 基本类同 JS 层的 resolve
        return FuflfillPromiseReactionJob(
            context, promiseOrCapability, result, reactionType);
    } catch (e) {
      // 基本类同 JS 层的 reject
      return RejectPromiseReactionJob(
          context, promiseOrCapability, e, reactionType);
    }
  }
}
```

PromiseReactionJob 接收的参数和 microtask 密切相关，argument 参数是 '123'，handler 是函数 () => {throw new Error('456')}，promiseOrCapability 是 p1，reactionType 是 kPromiseReactionFulfill。

handler 有值，进入 else 分支，在 try...catch 包裹下，试图调用 handler。handler 里 throw new Error('456') 抛出异常，被 catch 捕捉，调用 RejectPromiseReactionJob 方法，从函数名字也可以看出，p1 最终状态为 rejected。后面的代码和 JS 层面直接调用 reject 代码差不多，向 microtask 队列插入一个 microtask，这里不再赘述。当前 microtask 执行完毕后，会从 microtask 队列移除。

新增一个新 microtask，移除一个旧 microtask 后，microtask 队列简略示意图如下：

![promise2](https://raw.githubusercontent.com/xudale/blog/master/assets/p2.png)

handler 为 undefined 的原因是 p1 的最终状态是 rejected，但却没有 rejected 状态的处理函数。

开始执行下一个 microtask，还是调用上文提到的 PromiseReactionJob，argument 参数为 Error('456')，handler 是 undefined，promiseOrCapability 是 p2，reactionType 是 kPromiseReactionReject。由于 handler 是 undefined，这一次走的是 if 分支，最终调用了 RejectPromiseReactionJob，将 p2 状态置为 rejected。p1 相当于一个中转站，收到了 Error('456')，自己没有相应状态的处理函数，继续向下传给了 p2。执行完当前 microtask 后，microtask 队列的简略示意图如下：

![promise3](https://raw.githubusercontent.com/xudale/blog/master/assets/p3.png)

还是执行下一个 microtask，还是调用 PromiseReactionJob，argument 是 Error('456')，handler 是 (e) => console.log(e)，promiseOrCapability 是 p3，reactionType 是 kPromiseReactionReject。在 try...catch 中试图 handler，handler 不再抛异常，打印 Error('456')，返回 undefined。最后调用 FuflfillPromiseReactionJob，使 p3 最终状态是 fulfilled。执行完当前 microtask 后，microtask 队列的简略示意图如下：

![promise4](https://raw.githubusercontent.com/xudale/blog/master/assets/p4.png)

后面的流程和之前一样，就不解释了，上一个 microtask 的 handler (e) => console.log(e) 的返回值是 undefined，所以 (data) => console.log(data) 打印 undefined。

执行完所有 microtask 后，p0、p1、p2、p3 和 p4 状态如下，图是从浏览器控制台截的。

![5Promise](https://raw.githubusercontent.com/xudale/blog/master/assets/5Promise.png)

## 总结与感想

本文看似篇幅略长，其实大部分内容是 [Promise A+](https://promisesaplus.com/#notes) 规范的 2.2.7 节，规范简直字字珠玑。

![PromiseAPlus227](https://raw.githubusercontent.com/xudale/blog/master/assets/PromiseAPlus227.png)












