# Nodejs 如何获取 CPU 信息

基于 Mac，环境如下：
| 环境       |    版本 |
| --------  | -------- |
| Nodejs    | 12.16.1 |
| V8        | 7.8.279.23-node.31 |
| MacOS      | 10.13.5 |
| MacBook Pro      | 13-inch, Early 2011 |
| CPU      | 2.7 GHz Intel Core i7 |
| Xcode      | 9.4.1 |

Nodejs 获取 CPU 信息十分简单，加载 os 模块后，调用其 cpus 方法就可获取 CPU 信息，代码如下：

```JavaScript
const os = require('os');
const cpus = os.cpus()
console.log(cpus)
// 输出
[
  {
    model: 'Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz',
    speed: 2700,
    times: { user: 987260, nice: 0, sys: 859740, idle: 2834280, irq: 0 }
  },
  {
    model: 'Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz',
    speed: 2700,
    times: { user: 604020, nice: 0, sys: 288860, idle: 3624470, irq: 0 }
  },
  {
    model: 'Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz',
    speed: 2700,
    times: { user: 977040, nice: 0, sys: 584940, idle: 2955370, irq: 0 }
  },
  {
    model: 'Intel(R) Core(TM) i7-2620M CPU @ 2.70GHz',
    speed: 2700,
    times: { user: 600650, nice: 0, sys: 272840, idle: 3643860, irq: 0 }
  }
]
```

相关源码也很有层次性，从 JS -> C++ -> C -> 操作系统，下面逐层分析。

## JS
cpus 方法的源码位于 lib/os.js，粘贴如下：

```JavaScript
const {
  getCPUs
} = internalBinding('os');

function cpus() {
  // [] is a bugfix for a regression introduced in 51cea61
  const data = getCPUs() || [];
  const result = [];
  let i = 0;
  while (i < data.length) {
    result.push({
      model: data[i++],
      speed: data[i++],
      times: {
        user: data[i++],
        nice: data[i++],
        sys: data[i++],
        idle: data[i++],
        irq: data[i++]
      }
    });
  }
  return result;
}

module.exports = {
  cpus
};
```

cpus 通过调用 getCPUs 得到 data 数组，里面存储的是 CPU 相关的信息，data 数组的长度就是 CPU 的核心数。JS 层只是简单包装了一下 C++ 层的 getCPUs。
## C++
getCPUs 方法的源码位于 src/node_os.cc
## C
deps/uv/src/unix/darwin.c
## Mac OS

## Assembly
deps/v8/src/api/api.cc

## 核心逻辑
核心逻辑如下图，EventEmitter 的实例有属性 _events，_events 是一个对象，对象的 key 是事件名称，value 是函数或函数数组。EventEmitter 的所有方法几乎都围绕 _events 展开。
![eventEmitter](https://raw.githubusercontent.com/xudale/blog/master/assets/eventEmitter.png)

### EventEmitter 构造函数

EventEmitter 构造函数源码如下：
```JavaScript

#include <mach/mach.h>
#include <sys/sysctl.h>
#include <stdio.h>

int uv_cpu_info() {
    printf("uv_cpu_info\n");
    char model[512];
    uint64_t cpuspeed;
    size_t size;
    natural_t numcpus;
    mach_msg_type_number_t msg_type;
    processor_cpu_load_info_data_t *info;
    
    size = sizeof(model);
    sysctlbyname("machdep.cpu.brand_string", &model, &size, NULL, 0) &&
    sysctlbyname("hw.model", &model, &size, NULL, 0);
    
    printf("model is %s\n",model);
    
    size = sizeof(cpuspeed);
    sysctlbyname("hw.cpufrequency", &cpuspeed, &size, NULL, 0);
    printf("cpuspeed is %lld\n",cpuspeed);
    host_processor_info(mach_host_self(), PROCESSOR_CPU_LOAD_INFO, &numcpus,
                            (processor_info_array_t*)&info,
                        &msg_type);
    printf("numcpus is %u\n",numcpus);
    
    return 0;
}
void __cpuid(int cpu_info[4], int info_type) {
    __asm__ volatile("cpuid \n\t"
                     : "=a"(cpu_info[0]), "=b"(cpu_info[1]), "=c"(cpu_info[2]),
                     "=d"(cpu_info[3])
                     : "a"(info_type), "c"(0));
}
int main(int argc, const char * argv[]) {
    uv_cpu_info();
    int brand[12];
//    int cpu_info[4];
    __cpuid(&brand[0], 0x80000002);
    __cpuid(&brand[4], 0x80000003);
    __cpuid(&brand[8], 0x80000004);
    printf("Brand: %s\n", (char *)brand);
    return 0;
}
```

EventEmitter 构造函数的主要工作是初始化 _events，当实例自身没有 _events 属性时，执行 this._events = ObjectCreate(null)，ObjectGetPrototypeOf 实际上是 Object.getPrototypeOf 的别名，用于获取实例的原型。

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

逻辑很简单，读取 events[type]，判断是否注册过同名事件，如果没有则 events[type] = listener；如果之前注册过同名事件，把 listener 插在数组中。最后返回 this，以便链式调用。

### EventEmitter.prototype.emit

EventEmitter.prototype.emit 的源码如下：

```JavaScript
EventEmitter.prototype.emit = function emit(type, ...args) {
  // 这里的 ...args 用的很好，直接把函数的参数分成了 type 和 args，免于手动分离参数
  const events = this._events;
  const handler = events[type];

  if (handler === undefined)
    return false;

  if (typeof handler === 'function') {
    // 反射 API，ReflectApply 是 Reflect.apply 的别名
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
    }
  }

  return true;
};
```

删除了错误处理相关代码之后，逻辑很简单，根据事件名称 type，type 可以是字符串或 Symbol，从 _events 对象中取出要执行的函数或者函数数组。使用 ReflectApply 调用待触发的函数。ReflectApply 相当于 [Reflect.apply](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect/apply) 的别名。该方法与 ES5 中 Function.prototype.apply 方法类似。使用 Reflect.apply 方法会使代码更加简洁易懂。

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

off 方法和 on 方法源码除了逻辑相反外，整体结构类似；取 events[type]，如果是函数，直接 delete events[type]；如果是函数数组，则遍历数组，找到待删除的元素后，删除。

## 反射

从不同版本的 EventEmitter 源码来看，Reflect API 的使用频率有增多的趋势。[Reflect](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Reflect) 是一个内置的对象，它提供拦截 JavaScript 操作的方法，Reflect 对象提供了以下静态方法：

![reflect](https://raw.githubusercontent.com/xudale/blog/master/assets/reflect.png)

Reflect 方法与 Object、Function 上同名方法功能很类似，但使用 Reflect 会更加清晰优雅，而且把反射相关的方法收敛到 Reflect 对象，也与主流编程语言更接近。Reflect 方法在效率上应该是没有什么提升。比如本文使用的 Reflect.apply 方法，在 V8 中[源码如下](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/builtins/x64/builtins-x64.cc#1739)：

```c++
void Builtins::Generate_ReflectApply(MacroAssembler* masm) {
  {
    // __ 相当于 masm->
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

代码是 C++，硬是写出了 Intel 汇编的风格。这段代码在 V8 中的[宏处理](https://chromium.googlesource.com/v8/v8.git/+/refs/tags/7.8.279.23/src/builtins/setup-builtins-internal.cc#280)后会生成一个内置函数，这个内置函数最终会绑定在 Reflect.Apply 上。Generate_ReflectApply 的逻辑是取出 3 个参数，target、thisArgument、argumentsList，存在 3 个寄存器中，最后调用 CallWithArrayLike，CallWithArrayLike 方法才是 Reflect.apply 的核心逻辑，Generate_ReflectApply 只能算是一个入口。因为与汇编相关，Generate_ReflectApply 函数在不同平台有不同的实现。

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

逻辑和 Generate_ReflectApply 差不多，取 3 个参数放在 3 个寄存器，最后还是调用 CallWithArrayLike。所以 Reflect 对象的方法性能不见得比 Function 和 Object 的对应方法更强，只是可读性更高而已。

今天（2020.07.18）nodejs 的最新版本为 v14.5.0，把最新版本的 EventEmitter 源码与 v12.16.1 比较，可以看到 v14.5.0 把 Object.keys 替换为 Reflect.ownKeys

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
      // 有删减
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
      // 有删减
      return this;
    };
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

使用 Reflect.ownKeys 比 Object.keys 要更加合理。因为 Object.keys 返回的数组中没有 Symbol。

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

v14.5.0 的 EventEmitter 源码前十几行如下，从 primordials 导入当前文件要使用的 JavaScript 内置方法。

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

以笔者一个 nodejs 初学者的角度来看，ObjectDefineProperty、ObjectGetPrototypeOf 和 ObjectSetPrototypeOf 终究会被反射 API Reflect.defineProperty、Reflect.getPrototypeOf 和 Reflect.setPrototypeOf 方法所取代。

> 现阶段 Reflect 上的很多方法与 Object 相同，未来的明显属于语言内部的反射相关的方法将只部署在 Reflect 对象上

## 开枝散叶

EventEmitter 是很多类的父类，本节分析 EventEmitter 是如何子孙满堂的。笔者看了几处代码，写法都差不多，这里以 Stream 举例，Stream 构造函数源码如下：

```JavaScript
const EE = require('events');
function Stream(opts) {
  EE.call(this, opts);
}
ObjectSetPrototypeOf(Stream.prototype, EE.prototype);
ObjectSetPrototypeOf(Stream, EE);
```

Stream 继承 EventEmitter 的代码很简单，构造函数里面调用了 EventEmitter 的构造函数。分别设置了 Stream.prototype 和 Stream 的原型为 EE.prototype 和 EE，可以看出写法上已经极力向 class 写法靠拢。nodejs 源码很喜欢用比较新的 API 或 语法，笔者觉得未来有一天，这段代码可能会变成这样：

```JavaScript
const EE = require('events');
// 使用 class 后就不用手动设置那两个原型了
class Stream extends EE {
  constructor(opts) {
    super(opts)  
  }
}
```

笔者第一次见 JavaScript 的继承使用 Object.setPrototypeOf 设置原型时，总是感觉万一要写错，就认贼做父了。继承关系通常在项目构想阶段确定，JavaScript 运行时动态设置父类的写法看起来有些随意，笔者还是更喜欢和主流编程语言相近的 class 写法。

这里还可以期待下 JavaScript 的私有属性，在 nodejs 源码中，有的私有属性使用 _ 开头，如 _events，有的使用 Symbol。但两者都不是真正的私有属性，JavaScript 的私有属性长这样：

```JavaScript
class EventEmitter {
  #events = {}; // 私有属性以 # 开头
  on() {

  }
  off() {

  }
}
```

虽然丑，但感情和审美都是可以培养的，从 EventEmitter 源码来看，私有属性还是很有益处的，EventEmitter 并不希望调用者修改 _events，实际上调用者可以任意修改 _events。

> _ 开头的属性不是真正的私有属性，_ 只是它伪装成私有属性的保护色
> # 开头的属性，才是真正的私有属性



![private](https://raw.githubusercontent.com/xudale/blog/master/assets/private.png)

















