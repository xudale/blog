# EventEmitter 源码分析
EventEmitter 源码核心逻辑很简单，本文主要谈感想，版本为 nodejs 12.16.1，process.versions.v8 打印为 7.8.279.23-node.31。

## 核心逻辑
核心逻辑如下图，每个 EventEmitter 的实例有一个属性 _events，_events是一个对象，对象的 key 是事件名称，value 是函数或函数数组。几乎 EventEmitter 实例的所有方法都要操作 _events。
![eventEmitter](https://raw.githubusercontent.com/xudale/blog/master/assets/eventEmitter.png)

### EventEmitter 构造函数

EventEmitter 构造函数源码如下：
```JavaScript
function EventEmitter(opts) {
  EventEmitter.init.call(this, opts);
}

EventEmitter.init = function(opts) {
  if (this._events === undefined ||
      this._events === ObjectGetPrototypeOf(this)._events) {
    this._events = ObjectCreate(null);
    this._eventsCount = 0;
  }
};
```

EventEmitter 的构造函数实际上就是初始化 _events，当实例自身没有 _events 属性时，执行 this._events = ObjectCreate(null)，ObjectGetPrototypeOf 实际上相当于 Object.getPrototypeOf 的别名，用于获取实例原型链上是否存在 _events。

### EventEmitter.prototype.on

EventEmitter.prototype.on 的源码如下：

```JavaScript
EventEmitter.prototype.on = EventEmitter.prototype.addListener;

EventEmitter.prototype.addListener = function addListener(type, listener) {
  return _addListener(this, type, listener, false);
};

function _addListener(target, type, listener, prepend) {
  let m;
  let events;
  let existing;

  checkListener(listener);

  events = target._events;
  if (events === undefined) {
    events = target._events = ObjectCreate(null);
    target._eventsCount = 0;
  } else {
    // To avoid recursion in the case that type === "newListener"! Before
    // adding it to the listeners, first emit "newListener".
    if (events.newListener !== undefined) {
      target.emit('newListener', type,
                  listener.listener ? listener.listener : listener);

      // Re-assign `events` because a newListener handler could have caused the
      // this._events to be assigned to a new object
      events = target._events;
    }
    existing = events[type];
  }
  // 核心逻辑从这里开始
  if (existing === undefined) {
    // 之前未注册过同名事件
    // Optimize the case of one listener. Don't need the extra array object.
    // 只有一个 listener 的情况，直接添加
    events[type] = listener; 
    ++target._eventsCount;
  } else {
    // 之前注册同名事件
    if (typeof existing === 'function') {
      // Adding the second element, need to change to array.
      existing = events[type] =
        prepend ? [listener, existing] : [existing, listener];
      // If we've already got an array, just append.
    } else if (prepend) {
      existing.unshift(listener);
    } else {
      existing.push(listener);
    }
  }

  return target;
}
```

逻辑很简单，读取 events[type] 判断之前是否注册同名事件，如果没有则 events[type] = listener；如果之前注册过同名事件，把 listener 插在数组中。最后返回 this，方便链式调用。

### EventEmitter.prototype.emit

EventEmitter.prototype.emit 的源码如下：

```JavaScript
EventEmitter.prototype.emit = function emit(type, ...args) {
  // 这里的 ...args 用的很好，直接把函数的参数分成了 type 和 args，免去手动分离参数了
  const events = this._events;
  const handler = events[type];

  if (handler === undefined)
    return false;

  if (typeof handler === 'function') {
    const result = ReflectApply(handler, this, args);

    // We check if result is undefined first because that
    // is the most common case so we do not pay any perf
    // penalty
    if (result !== undefined && result !== null) {
      addCatch(this, result, type, args);
    }
  } else {
    const len = handler.length;
    const listeners = arrayClone(handler, len);
    for (let i = 0; i < len; ++i) {
      const result = ReflectApply(listeners[i], this, args);

      // We check if result is undefined first because that
      // is the most common case so we do not pay any perf
      // penalty.
      // This code is duplicated because extracting it away
      // would make it non-inlineable.
      if (result !== undefined && result !== null) {
        addCatch(this, result, type, args);
      }
    }
  }

  return true;
};
```

删除了错误处理相关代码之后，逻辑很简单，根据事件名称 type，type 可以是字符串或 Symbol，从 _events 对象中取出要执行的函数或者函数数组，对于触发的函数，使用 ReflectApply(listeners[i], this, args) 调用。ReflectApply 相当于 [Reflect.apply](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect/apply) 的别名。

### EventEmitter.prototype.off

EventEmitter.prototype.off 的源码如下：

```JavaScript
EventEmitter.prototype.off = EventEmitter.prototype.removeListener;

EventEmitter.prototype.removeListener =
    function removeListener(type, listener) {
      let originalListener;

      checkListener(listener);

      const events = this._events;

      const list = events[type];
      if (list === undefined)
        return this;

      if (list === listener || list.listener === listener) {
        if (--this._eventsCount === 0)
          this._events = ObjectCreate(null);
        else {
          delete events[type];
          if (events.removeListener)
            this.emit('removeListener', type, list.listener || listener);
        }
      } else if (typeof list !== 'function') {
        let position = -1;

        for (let i = list.length - 1; i >= 0; i--) {
          if (list[i] === listener || list[i].listener === listener) {
            originalListener = list[i].listener;
            position = i;
            break;
          }
        }

        if (position < 0)
          return this;

        if (position === 0)
          list.shift();
        else {
          if (spliceOne === undefined)
            spliceOne = require('internal/util').spliceOne;
          spliceOne(list, position); // spliceOne 类似 splice 方法，但在当前业务场景下效率更高些
        }

        if (list.length === 1)
          events[type] = list[0];
      }

      return this;
    };
```

off 方法和 on 方法源码除了逻辑相反外，看起来差不多；取 events[type]，如果是函数，直接 delete events[type]；如果是函数数组，则遍历数组，找到待删除的元素删除。

## 反射

从不同版本的 EventEmitter 的源码来看，反射用的越来越多，[Reflect](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect) 是一个内置的对象，它提供拦截 JavaScript 操作的方法，对象提供了以下静态方法：

![reflect](https://raw.githubusercontent.com/xudale/blog/master/assets/reflect.png)

Reflect 方法与 Object、Function 上同名方法功能很类似，使用 Reflect 会更加清晰优雅，而且把反射相关的方法收敛到 Reflect 对象，也与主流编程语言更接近。Reflect 方法在效率上应该是没有什么提升。比如本文使用的 Reflect.apply 方法，在 V8 中[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/builtins/x64/builtins-x64.cc#1739)：

```c++
void Builtins::Generate_ReflectApply(MacroAssembler* masm) {
  {
    Label done;
    StackArgumentsAccessor args(rsp, rax);
    __ LoadRoot(rdi, RootIndex::kUndefinedValue);
    __ movq(rdx, rdi);
    __ movq(rbx, rdi);
    __ cmpq(rax, Immediate(1));
    __ j(below, &done, Label::kNear);
    __ movq(rdi, args.GetArgumentOperand(1));  // target
    __ j(equal, &done, Label::kNear);
    __ movq(rdx, args.GetArgumentOperand(2));  // thisArgument
    __ cmpq(rax, Immediate(3));
    __ j(below, &done, Label::kNear);
    __ movq(rbx, args.GetArgumentOperand(3));  // argumentsList
    __ bind(&done);
    __ PopReturnAddressTo(rcx);
    __ leaq(rsp,
            Operand(rsp, rax, times_system_pointer_size, kSystemPointerSize));
    __ Push(rdx);
    __ PushReturnAddressFrom(rcx);
  }
  // 3. Apply the target to the given argumentsList.
  __ Jump(BUILTIN_CODE(masm->isolate(), CallWithArrayLike),
          RelocInfo::CODE_TARGET);
}
```

代码是 C++ 代码，风格是 X64 汇编的风格。这段代码在 V8 中的[宏处理](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/builtins/setup-builtins-internal.cc#280)后会生成一个内置函数。Generate_ReflectApply 的逻辑是取出 3 个参数，target、thisArgument、argumentsList 放在 3 个寄存器，最后调用 CallWithArrayLike，CallWithArrayLike 方法才是 Reflect.apply 的核心逻辑，Generate_ReflectApply 只能算是一个入口。Generate_ReflectApply 函数在不同平台有不同的汇编实现。

Function.prototype.apply 的[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/builtins/x64/builtins-x64.cc#1622):

```c++
void Builtins::Generate_FunctionPrototypeApply(MacroAssembler* masm) {
  {
    Label no_arg_array, no_this_arg;
    StackArgumentsAccessor args(rsp, rax);
    __ LoadRoot(rdx, RootIndex::kUndefinedValue);
    __ movq(rbx, rdx);
    __ movq(rdi, args.GetReceiverOperand());
    __ testq(rax, rax);
    __ j(zero, &no_this_arg, Label::kNear);
    {
      __ movq(rdx, args.GetArgumentOperand(1));
      __ cmpq(rax, Immediate(1));
      __ j(equal, &no_arg_array, Label::kNear);
      __ movq(rbx, args.GetArgumentOperand(2));
      __ bind(&no_arg_array);
    }
    __ bind(&no_this_arg);
    __ PopReturnAddressTo(rcx);
    __ leaq(rsp,
            Operand(rsp, rax, times_system_pointer_size, kSystemPointerSize));
    __ Push(rdx);
    __ PushReturnAddressFrom(rcx);
  }
  Label no_arguments;
  __ JumpIfRoot(rbx, RootIndex::kNullValue, &no_arguments, Label::kNear);
  __ JumpIfRoot(rbx, RootIndex::kUndefinedValue, &no_arguments, Label::kNear);

  // 4a. Apply the receiver to the given argArray.
  __ Jump(BUILTIN_CODE(masm->isolate(), CallWithArrayLike),
          RelocInfo::CODE_TARGET);

  // 4b. The argArray is either null or undefined, so we tail call without any
  // arguments to the receiver. Since we did not create a frame for
  // Function.prototype.apply() yet, we use a normal Call builtin here.
  __ bind(&no_arguments);
  {
    __ Set(rax, 0);
    __ Jump(masm->isolate()->builtins()->Call(), RelocInfo::CODE_TARGET);
  }
}
```

逻辑和 Generate_ReflectApply 差不多，取 3 个参数放在 3 个寄存器，最后调用 CallWithArrayLike。所以 Reflect 对象的方法性能不见得比 Function 和 Object 的对应方法性能更优秀，只是可读性更高而已。

今天（2020.07） nodejs 的最新版本 v14.5.0 的 EventEmitter 源码与本文的版本 v12.16.1 比较，可以看到 v14.5.0 把 Object.keys 替换为 Reflect.keys：

```c++
// v12.16.1
EventEmitter.prototype.removeAllListeners =
    function removeAllListeners(type) {
      const events = this._events;
      // Emit removeListener for all listeners on all events
      if (arguments.length === 0) {
        // 这里用 Object.keys 遍历 events
        for (const key of ObjectKeys(events)) {
          if (key === 'removeListener') continue;
          this.removeAllListeners(key);
        }
        this.removeAllListeners('removeListener');
        this._events = ObjectCreate(null);
        this._eventsCount = 0;
        return this;
      }

      const listeners = events[type];

      if (typeof listeners === 'function') {
        this.removeListener(type, listeners);
      } else if (listeners !== undefined) {
        // LIFO order
        for (let i = listeners.length - 1; i >= 0; i--) {
          this.removeListener(type, listeners[i]);
        }
      }

      return this;
    };
```

```c++
// v14.5.0
EventEmitter.prototype.removeAllListeners =
    function removeAllListeners(type) {
      const events = this._events;
      // Emit removeListener for all listeners on all events
      if (arguments.length === 0) {
        // 最新版本用 Reflect.ownKeys 遍历 events
        for (const key of ReflectOwnKeys(events)) {
          if (key === 'removeListener') continue;
          this.removeAllListeners(key);
        }
        this.removeAllListeners('removeListener');
        this._events = ObjectCreate(null);
        this._eventsCount = 0;
        return this;
      }

      const listeners = events[type];

      if (typeof listeners === 'function') {
        this.removeListener(type, listeners);
      } else if (listeners !== undefined) {
        // LIFO order
        for (let i = listeners.length - 1; i >= 0; i--) {
          this.removeListener(type, listeners[i]);
        }
      }

      return this;
    };
```

使用 Reflect.ownKeys 比 Object.keys 要更合理，因为 Object.keys 返回的数组中没有 Symbol。

```JavaScript
let sy = Symbol('test')
let obj = {
  a: 1,
  b: 2,
  [sy]: 'symbol'
}

Object.keys(obj) // ["a", "b"]
Reflect.ownKeys(obj) // ["a", "b", Symbol(test)]

```

考虑到事件名称可以是 Symbol，比如：

```JavaScript
const EventEmitter = require('events');
class MyEmitter extends EventEmitter {}
const myEmitter = new MyEmitter();
// 这里用 Symbol 当做事件名称
let sy = Symbol('abc')
myEmitter.on(sy, () => {
  console.log('Symbol event occured');
});
myEmitter.emit(sy);
```

v14.5.0 的 EventEmitter 源码前十几行是这样的：

```JavaScript
const {
  Boolean,
  Error,
  MathMin,
  NumberIsNaN,
  ObjectCreate,
  ObjectDefineProperty,
  ObjectGetPrototypeOf,
  ObjectSetPrototypeOf,
  Promise,
  PromiseReject,
  PromiseResolve,
  ReflectApply,
  ReflectOwnKeys,
  Symbol,
  SymbolFor,
  SymbolAsyncIterator
} = primordials;
```

从 nodejs 不同版本 EventEmitter 源码来看，ObjectDefineProperty、ObjectGetPrototypeOf 和 ObjectSetPrototypeOf 早晚会被 Reflect.defineProperty、Reflect.getPrototypeOf 和 Reflect.setPrototypeOf 方法所取代。


## 开枝散叶

EventEmitter 是很多类的父类，本小节分析 EventEmitter 是如何喜当爹的。笔者看了几处代码，写法都差不多，这里以 Stream 举例，Stream 构造函数源码如下：

```JavaScript
const EE = require('events');
function Stream(opts) {
  EE.call(this, opts);
}
ObjectSetPrototypeOf(Stream.prototype, EE.prototype);
ObjectSetPrototypeOf(Stream, EE);
```

Stream 继承 EventEmitter 的代码很简单，构造函数里面先是调用了 EventEmitter 的构造函数。分别设置了 Stream.prototype 和 Stream 的原型为 EE.prototype 和 EE。可以看出写法上已经极力向 class 写法靠拢。nodejs 源码很喜欢用比较新的 API 或 语法，笔者觉得未来有一天，这段代码可能会变成这样：

```JavaScript
const EE = require('events');
class MyStream extends EE {

}
```

笔者是半路出家做的前端，第一次见 JavaScript 的继承 Object.setPrototypeOf 的写法时，总是感觉万一要是写错了，就认贼做父了。而且继承这件事情通常在代码还没运行前的项目构想阶段就已经确定了，JavaScript 动态设置父类的写法总是有点奇怪。笔者还是更喜欢 class 写法。

## 简易版 EventEmitter

```JavaScript
class MyEventEmitter {
  constructor() {
    this._events = Object.create(null)
  }
  on(type, listener) {
    this._events[type] = this._events[type] || []
    this._events[type].push(listener)
    return this
  }
  off(type, listener) {
    this._events[type] = this._events[type].filter(current => current !== listener)
    return this
  }
  emit(type, ...args) {
    this._events[type].forEach(current => {
      Reflect.apply(current, this, args)
    })
  }
}
```

MyEventEmitter 不到 20 行，只有基本逻辑，没有做任何参数校验，不能用于实际生产环境。实现参数校验、异常情况、旧版本兼容和性能调优的代码量要 10 倍于基本逻辑的代码量。















